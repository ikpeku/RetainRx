# Data Model

PostgreSQL 16 via Drizzle. This document is the contract; `packages/db/schema/*.ts` must match it. Any change to the schema updates this file in the same PR.

Conventions:
- Primary keys are `uuid` (v7, time-ordered) named `id`.
- Every tenant table has `pharmacy_id uuid NOT NULL REFERENCES pharmacies(id)`.
- `created_at timestamptz NOT NULL DEFAULT now()` and `updated_at` on mutable tables.
- Money is `bigint` **kobo** (₦1 = 100 kobo), with a column suffix `_kobo`.
- Business dates (`due_date`) are `date` in Africa/Lagos. Instants are `timestamptz`.
- Enums are Postgres enums, listed below.
- Soft delete is **not** used for personal data. Erasure is real deletion or anonymisation (see [security-and-privacy.md](security-and-privacy.md)).

## Entity overview

```
pharmacies ─┬─< staff_roles >── staff
            ├─< devices
            ├─< medicines ─┬─< stock_checks
            │              └─< stock_requests >── customers
            ├─< customers ─┬─< consents
            │              ├── conversations (1:1)
            │              ├─< medication_plans ──< refill_cycles ──< scheduled_messages
            │              └─< messages
            ├─< stock_queries
            ├─< counter_actions
            ├─< clinical_flags
            ├─< restock_notifications
            ├─< measurement_snapshots
            └─< invoices
events (append-only, all domains)
```

## Tables

### pharmacies
| column | type | notes |
|---|---|---|
| id | uuid | |
| name | text | "Ada Pharmacy" |
| join_code | text UNIQUE | e.g. `ADA-4821`, used in the QR |
| status | `pharmacy_status` | `onboarding` → `verified` → `active` → `suspended` / `churned` |
| pcn_premises_licence | text | verified at onboarding |
| address, lga, state | text | |
| geo | point NULL | |
| default_refill_value_kobo | bigint | fallback for revenue calculations |
| reminder_local_time | time | default `10:00` |
| control_share | numeric(3,2) | default `0.20` |
| stock_check_freshness_minutes | int | default `120` |
| pricing_plan_id | uuid FK | |
| verified_at, verified_by | timestamptz, uuid | |

### staff / staff_roles
| staff column | type | notes |
|---|---|---|
| id | uuid | |
| phone_e164 | text UNIQUE | login identifier |
| full_name | text | |
| pcn_registration_no | text NULL | required for superintendent |

`staff_roles`: (`staff_id`, `pharmacy_id`, `role` `staff_role` = `owner` \| `superintendent`), PK on all three. One person can hold both roles.

### devices (counter tablets)
| column | type | notes |
|---|---|---|
| id, pharmacy_id | uuid | |
| label | text | "Front counter" |
| token_hash | text UNIQUE | sha256(token + pepper). The raw token is shown once only |
| paired_at, last_seen_at, revoked_at | timestamptz | |

`device_pairing_codes`: (`code` 6 digits, `pharmacy_id`, `expires_at` +10 min, `used_at`).

### customers
| column | type | notes |
|---|---|---|
| id, pharmacy_id | uuid | UNIQUE(pharmacy_id, phone_e164) |
| phone_e164 | text | |
| display_name | text NULL | confirmed during consent |
| status | `customer_status` | `pending_consent` → `active` → `paused` / `withdrawn` |
| experiment_arm | `experiment_arm` | `on_time` \| `control` \| `excluded`. Assigned at first enrolment and **immutable** except → `excluded` by clinical exemption |
| arm_assigned_at | timestamptz | |
| enrolled_via | text | `qr` \| `counter` |

### consents
Append-only. The current consent is the latest row.
| column | type | notes |
|---|---|---|
| id, pharmacy_id, customer_id | uuid | |
| action | `consent_action` | `granted` \| `updated` \| `withdrawn` |
| version | text | consent text version, e.g. `2026-10-v1` |
| scopes | text[] | `reminders`, `stock_alerts`, `medication_history` |
| wa_message_id | text | the button reply that gave consent |
| created_at | timestamptz | |

### conversations
| column | type | notes |
|---|---|---|
| customer_id | uuid PK | one per customer (per pharmacy) |
| state | text | Rexa state machine node, e.g. `capture.awaiting_duration` |
| context | jsonb | draft plan, pending clarification, retry counts |
| last_inbound_at | timestamptz | drives the 24h free-form window |
| flagged_for_human_at | timestamptz NULL | "Talk to the pharmacy" |

### medicines (pharmacy catalogue)
| column | type | notes |
|---|---|---|
| id, pharmacy_id | uuid | |
| name | text | generic or brand, e.g. "Amlodipine" |
| strength | text | "5mg" |
| form | text | tablet, capsule, syrup, injection… |
| barcode | text NULL | indexed |
| unit_price_kobo | bigint NULL | optional; used for refill value |
| normalized_key | text | lowercase name+strength for matching; UNIQUE(pharmacy_id, normalized_key) |

Medicines are auto-created from enrolments and stock queries when they don't exist yet, with `source='customer'`.

### medication_plans
| column | type | notes |
|---|---|---|
| id, pharmacy_id, customer_id, medicine_id | uuid | |
| quantity | int | 30 |
| quantity_unit | text | "tablets" |
| duration_days | int | 30 for "1 month" |
| status | `plan_status` | `active` \| `paused` \| `stopped` |
| source_photo_key | text NULL | R2 key; deleted after 30 days |
| extraction_confidence | numeric NULL | 0–1 |
| clinical_hold | boolean | set by superintendent to block reminders |

### refill_cycles
| column | type | notes |
|---|---|---|
| id, pharmacy_id, plan_id | uuid | |
| seq | int | 1, 2, 3… UNIQUE(plan_id, seq) |
| supply_start | date | |
| due_date | date | supply_start + duration_days |
| arm | `experiment_arm` | copied from customer at creation (frozen for measurement) |
| reminder_due_at | timestamptz | when the reminder is scheduled for this cycle's arm |
| ontime_reminder_at | timestamptz | due_date − 3d @ reminder_local_time, **for both arms**. Anchors the measurement window |
| refill_requested_at | timestamptz NULL | customer replied "Yes" |
| outcome | `cycle_outcome` NULL | `collected` \| `declined` \| `unavailable` \| `stopped` \| `superseded` |
| outcome_at | timestamptz NULL | |
| collected_at | timestamptz NULL | |
| refill_value_kobo | bigint NULL | snapshot at collection |

Partial unique index: one open cycle per plan (`WHERE outcome IS NULL OR outcome = 'unavailable'`).

### scheduled_messages
| column | type | notes |
|---|---|---|
| id, pharmacy_id, customer_id | uuid | |
| kind | `scheduled_kind` | `refill_reminder` \| `back_in_stock` \| `stock_query_followup` |
| cycle_id | uuid NULL | |
| send_at | timestamptz | |
| status | `scheduled_status` | `pending` \| `queued` \| `sent` \| `cancelled` \| `vetoed` \| `failed` |
| vetoed_by, veto_reason | uuid, text | superintendent |
| message_id | uuid NULL | FK to messages once sent |

### messages
| column | type | notes |
|---|---|---|
| id, pharmacy_id NULL, customer_id NULL | uuid | pharmacy is NULL before the join code is resolved |
| direction | `in` \| `out` | |
| wa_message_id | text UNIQUE | |
| kind | text | text, image, interactive, template |
| template_name | text NULL | |
| body_redacted | text | short preview for support; full body is **not** retained beyond 90 days |
| payload | jsonb | raw (90-day retention) |
| status | text | received, sent, delivered, read, failed |
| error_code | text NULL | |

### counter_actions
| column | type | notes |
|---|---|---|
| id, pharmacy_id, device_id | uuid | |
| client_action_id | uuid UNIQUE | idempotency from the offline queue |
| action | `counter_action` | `collected` \| `didnt_have` |
| plan_id, cycle_id | uuid | |
| undone_at | timestamptz NULL | within 30s |

### stock_checks
| column | type | notes |
|---|---|---|
| id, pharmacy_id, medicine_id | uuid | |
| status | `stock_status` | `in_stock` \| `low` \| `out` |
| checked_by_device_id / checked_by_staff_id | uuid NULL | |
| stock_query_id | uuid NULL | if answering a WhatsApp query |
| checked_at | timestamptz | |

### stock_queries (WhatsApp "Do you have this?")
| column | type | notes |
|---|---|---|
| id, pharmacy_id, customer_id | uuid | |
| medicine_id | uuid NULL | NULL if not matched; raw text kept |
| raw_text | text | |
| status | `pending` \| `answered` \| `timed_out` | |
| answered_at | timestamptz | used for the 2–10 min SLA metric |

### stock_requests ("Asked for")
| column | type | notes |
|---|---|---|
| id, pharmacy_id, medicine_id | uuid | |
| customer_id | uuid NULL | NULL for anonymous walk-ins |
| source | `whatsapp` \| `counter` | |
| status | `request_status` | `requested` → `in_transit` → `arrived` → `notified` → `fulfilled`; or `expired` (60 days) / `cancelled` |
| cycle_id | uuid NULL | if from "Didn't have it" on a refill |
| requested_at, arrived_at, notified_at, fulfilled_at | timestamptz | |

### restock_notifications
| column | type | notes |
|---|---|---|
| id, pharmacy_id, medicine_id | uuid | |
| marked_in_stock_by | uuid | staff |
| marked_at, notified_at | timestamptz | |
| recipients_count | int | |

### clinical_flags
| column | type | notes |
|---|---|---|
| id, pharmacy_id, customer_id | uuid | |
| plan_id | uuid NULL | |
| kind | `alert` \| `hold_reminders` \| `exclude_from_experiment` \| `message_review` | |
| note | text | visible to superintendent and owner |
| created_by, resolved_by | uuid | |
| resolved_at | timestamptz NULL | |

### measurement_snapshots
One row per pharmacy per period (month), recomputed nightly until the period closes.
| column | type | notes |
|---|---|---|
| id, pharmacy_id | uuid | UNIQUE(pharmacy_id, period_start) |
| period_start, period_end | date | |
| n_ontime, returns_ontime | int | |
| n_control, returns_control | int | |
| rate_ontime, rate_control, lift | numeric | |
| lift_ci_low, lift_ci_high | numeric | 95% Wilson/Newcombe |
| lift_source | `pharmacy` \| `network_pooled` | |
| incremental_returns | numeric | |
| avg_refill_value_kobo | bigint | |
| incremental_revenue_kobo | bigint | |
| status | `provisional` \| `final` | final once the period's last window closes |

### pricing_plans / invoices
`pricing_plans`: `id`, `name`, `kind` (`flat` \| `performance` \| `hybrid`), `flat_fee_kobo`, `performance_pct` (e.g. 0.10), `min_fee_kobo`, `max_fee_kobo`.

`invoices`: `id`, `pharmacy_id`, `snapshot_id`, `number` (`RRX-2026-000123`), `amount_kobo`, `line_items` jsonb, `status` (`draft` \| `issued` \| `paid` \| `void`), `paystack_reference`, `issued_at`, `due_at`, `paid_at`, `pdf_key`.

### events (append-only audit log)
| column | type | notes |
|---|---|---|
| id | uuid | |
| pharmacy_id | uuid NULL | |
| actor_type | `customer` \| `device` \| `staff` \| `admin` \| `system` | |
| actor_id | uuid NULL | |
| type | text | dotted, e.g. `consent.granted`, `plan.created`, `reminder.sent`, `refill.collected`, `stock.request_created`, `restock.notified`, `clinical.veto` |
| subject_type, subject_id | text, uuid | |
| data | jsonb | no raw PII |
| occurred_at | timestamptz | |

No UPDATE or DELETE grants on `events` for the app role (except anonymisation on erasure).

## Derived values (in `packages/domain`, never stored)

- `cycleStatus(cycle, todayLagos)` → `upcoming | due | overdue | lapsed | closed`
- `isReturn(cycle)` → `collected_at` within `[ontime_reminder_at, ontime_reminder_at + 7d)`. See [measurement-and-billing.md](measurement-and-billing.md).
- `refillValue(plan)` → `medicine.unit_price_kobo × quantity` if price set, else `pharmacy.default_refill_value_kobo`.

## Indexes (minimum)

- `refill_cycles (pharmacy_id, due_date) WHERE outcome IS NULL`, for due/overdue/lapsed lists
- `scheduled_messages (status, send_at)`, for the dispatcher
- `stock_requests (pharmacy_id, medicine_id, status)`
- `stock_checks (pharmacy_id, medicine_id, checked_at DESC)`
- `messages (customer_id, created_at DESC)`
- `customers (pharmacy_id, phone_e164)` unique; trigram index on `display_name` for counter search
