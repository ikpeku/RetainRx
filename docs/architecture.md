# Architecture

Related: [tech-stack.md](tech-stack.md) · [data-model.md](data-model.md) · [api-spec.md](api-spec.md) · [conversation-design.md](conversation-design.md)

## 1. System context

```
                       ┌────────────────────────────┐
  Customer ──WhatsApp──▶  Meta WhatsApp Cloud API   │
                       └──────────┬─────────────────┘
                         webhook  │  ▲ send (Graph API)
                                  ▼  │
┌──────────────┐  HTTPS   ┌───────────────────────┐   SQL   ┌──────────────┐
│ Counter Panel│─────────▶│                       │────────▶│              │
│  (PWA, tab)  │          │     apps/api          │         │  PostgreSQL  │
└──────────────┘          │  Fastify REST +       │◀────────│  (single     │
┌──────────────┐  HTTPS   │  webhooks             │         │  source of   │
│Owner Workspace│────────▶│                       │         │  truth +     │
│Clinical page │          └───────────┬───────────┘         │  pg-boss     │
│Admin console │                      │ enqueue jobs        │  queue)      │
│ (apps/web)   │                      ▼                     │              │
└──────────────┘          ┌───────────────────────┐   SQL   │              │
                          │     apps/worker       │────────▶│              │
                          │ reminders · notifies  │         └──────────────┘
                          │ measurement · invoices│
                          └──┬─────────┬──────────┘
                             │         │
                 Claude API ◀┘         └▶ Paystack · Termii · R2
```

- **apps/web** never talks to the database directly. It calls **apps/api** (server components call it server-side, client components use TanStack Query). This keeps a single authorization layer.
- **apps/api** owns all reads and writes, verifies webhooks, and enqueues jobs. It does no slow work inline. Anything over ~300ms (AI extraction, bulk notifications) is a job.
- **apps/worker** consumes pg-boss queues and runs crons.
- **Postgres** holds both domain data and the job queue, so a domain write and its job enqueue happen in **one transaction** (transactional outbox via pg-boss in the same connection).

## 2. Mapping the product to components

| Product area (from the design) | Component | Packages |
|---|---|---|
| 1. Customer Journey (Rexa on WhatsApp) | `api` webhook → `rexa` engine → `whatsapp` client | `rexa`, `whatsapp`, `ai`, `domain` |
| Other customer interactions | Same engine, intent router | `rexa`, `ai` |
| 2. Pharmacy Counter Panel | `web` `(counter)` route group (PWA) + `api` `/counter/*` | `ui`, `domain` |
| When a medicine is unavailable | `api` stock queries / requests + `worker` notifications | `domain`, `rexa` |
| 3. Owner Workspace + Clinical page | `web` `(owner)` and `(clinical)` + `api` `/workspace/*`, `/clinical/*` | `ui`, `domain` |
| Measurement & Billing | `worker` nightly/monthly jobs + `api` `/workspace/reports` | `domain` (metrics, pricing) |
| Backend (single source of truth) | Postgres + `db` | `db` |
| Support systems | `web` `(admin)` + `api` `/admin/*` | `ui` |

## 3. Backend data domains

These follow the five boxes in the design. Full schema is in [data-model.md](data-model.md).

| Domain | Tables |
|---|---|
| **Customer data** (contacts, consent, profile) | `customers`, `consents`, `conversations` |
| **Medication data** (name, dose, duration) | `medicines`, `medication_plans`, `refill_cycles` |
| **Pharmacy data** (stock, staff, location) | `pharmacies`, `staff`, `staff_roles`, `devices`, `stock_checks`, `stock_queries` |
| **Events & actions** (reminders, requests, returns) | `messages`, `scheduled_messages`, `counter_actions`, `stock_requests`, `restock_notifications`, `clinical_flags`, `events` |
| **Analytics** (measurement, billing) | `experiment_assignments` (on `customers`), `measurement_snapshots`, `invoices`, `pricing_plans` |

## 4. Key flows

### 4.1 Inbound WhatsApp message

```
Meta ──POST /webhooks/whatsapp──▶ api
  1. Verify X-Hub-Signature-256 with WHATSAPP_APP_SECRET (reject on mismatch)
  2. Insert into messages (direction=in, wa_message_id UNIQUE) → duplicate? return 200, stop
  3. Enqueue job `rexa.handle_inbound` {message_id}   (same txn)
  4. Return 200 immediately (Meta retries if not 200 within ~seconds)

worker: rexa.handle_inbound
  1. Load conversation (customer + pharmacy + state) with SELECT … FOR UPDATE
  2. If image: download media → R2 → ai.extractMedicine() (Claude vision, tool-use schema)
  3. If text and not a button reply: ai.classifyIntent() (Haiku) → intent + slots
  4. rexa.step(state, input) → {nextState, outbound[], domainCommands[]}   ← pure function
  5. Apply domainCommands (create plan, record consent, create stock query…) + write events
  6. Persist nextState, enqueue `whatsapp.send` for each outbound message
```

`rexa.step` is a **pure function** (state + input → state + effects). It is unit-tested exhaustively without WhatsApp or AI.

### 4.2 Reminder scheduling

```
On enrolment or on "Collected refill":
  domain.startCycle(plan, supplyStart) →
     due_date        = supplyStart + duration_days
     reminder_at     = arm == on_time ? due_date − 3d : due_date + 4d   (at 10:00 WAT)
  INSERT refill_cycles + scheduled_messages(kind=reminder, send_at=reminder_at)

cron every 5 min: `reminders.dispatch`
  SELECT scheduled_messages WHERE send_at <= now() AND status='pending'
         AND not vetoed AND plan active AND consent active
         FOR UPDATE SKIP LOCKED LIMIT 200
  → enqueue whatsapp.send(template=refill_reminder_v1) ; status='queued'
  → if outside 09:00–19:00 WAT, push to next 10:00 WAT
```

### 4.3 Counter "Collected refill"

```
POST /counter/actions {plan_id, action:'collected', client_action_id}
  → idempotent on client_action_id (offline queue may retry)
  → close current cycle (outcome=collected, collected_at=now)
  → cancel its pending scheduled reminder if not yet sent
  → start next cycle with supplyStart = today
  → events: refill.collected
```

### 4.4 Stock query from WhatsApp ("Do you have X?")

```
rexa intent=stock_query(medicine_text)
  → match to pharmacy medicines (fuzzy + AI normalisation)
  → latest stock_check for that medicine ≤ 2h old?  yes → answer now ("As of 2h ago: in stock")
                                                   no  → create stock_query(status=pending), reply "Let me check with the pharmacy…"
Counter Panel polls /counter/stock-queries (every 15s) → attendant answers
  → stock_check inserted; stock_query answered; whatsapp.send(answer)
  → if out of stock: stock_request created (Asked for) + Rexa offers "Want me to tell you when it arrives?"
cron: stock query older than 10 min and unanswered → Rexa: "The pharmacy is busy, I'll message you as soon as they check."; flag in workspace
```

### 4.5 Restock notification

```
Owner taps "Now in stock" (OW-4) for medicine M
  → stock_requests where medicine=M and status in (requested,in_transit) → status=arrived
  → Notify (immediately or via button) → enqueue whatsapp.send(template=back_in_stock_v1) per customer
  → status=notified, restock_notifications row records count + time
```

## 5. Multi-tenancy

- Every tenant-owned table has `pharmacy_id NOT NULL`.
- The API resolves the tenant from the session (staff), the device token (counter), or the conversation (Rexa). **Never from a request parameter.**
- All repository functions take `pharmacyId` as their first argument. A lint rule plus tests assert that no tenant query is missing it.
- Postgres Row-Level Security is enabled as defence-in-depth: the API sets `app.pharmacy_id` per transaction.
- Customers are **scoped per pharmacy**. The same phone number enrolled at two pharmacies is two `customers` rows. The Rexa conversation picks the active pharmacy from the most recent join code, or asks if ambiguous.

## 6. Time and scheduling rules

- Store timestamps as `timestamptz` (UTC). Store business dates (`due_date`) as `date`, computed in `Africa/Lagos`.
- "Today" for status buckets is always the Lagos calendar date.
- Status buckets are **derived** (never stored), via `domain.cycleStatus(cycle, today)`:
  - `upcoming`: due_date > today + 2
  - `due`: today ≤ due_date ≤ today + 2
  - `overdue`: today − 7 ≤ due_date < today
  - `lapsed`: due_date < today − 7 and not collected
  - `closed`: outcome set (collected / declined / stopped)

## 7. Reliability

- **Idempotency:** inbound `wa_message_id` is unique. Outbound sends carry `scheduled_message_id` and are not resent once Meta returns a message ID. Counter actions use `client_action_id`.
- **Retries:** pg-boss exponential backoff (max 5) for sends. Permanent WhatsApp errors (e.g. 131026 undeliverable) mark the message failed and surface it in the admin console.
- **Delivery status:** Meta status webhooks (`sent`, `delivered`, `read`, `failed`) update `messages.status`.
- **24-hour window:** free-form messages only inside 24h of the customer's last inbound message. Otherwise an approved template is used. Enforced in `packages/whatsapp` (it refuses to send free-form outside the window).

## 8. Observability

- pino JSON logs with `request_id`, `pharmacy_id`, `job_id`. Phone numbers and message bodies are **redacted**.
- Sentry for errors in api, worker and web.
- Admin console health panel: messages sent/failed per day, pending queue depth, unanswered stock queries, WhatsApp quality rating.

## 9. Security boundaries

See [security-and-privacy.md](security-and-privacy.md). In short: four auth contexts (staff session, device token, admin OAuth, Meta/Paystack webhook signature), role checks in the API, RLS, PII redaction, and encryption at rest by the provider.
