# Implementation Plan

The build is ordered so that each phase ends with something **demoable end-to-end**. Task IDs (`P2.3`) are used in commits, PRs and [progress.md](progress.md). Requirement IDs (`CJ-3`) refer to [prd.md](prd.md).

**How to work this plan (humans and AI agents):**
1. Pick the first unchecked task in [progress.md](progress.md) whose dependencies are done.
2. Read the referenced spec sections before writing code.
3. Implement it with tests. Meet **every** acceptance criterion.
4. Update [progress.md](progress.md) (checkbox + log line). Update the specs if reality differed.
5. One task per PR where practical. PR title: `P2.3: Capture medication via photo`.

---

## Phase 0: Foundations
Goal: an empty but production-shaped monorepo that deploys.

| ID | Task | Acceptance criteria |
|---|---|---|
| P0.1 | Scaffold monorepo (pnpm, Turborepo, `apps/{api,worker,web}`, `packages/{db,domain,rexa,whatsapp,ai,ui,config}`), strict TS, ESLint, Prettier | `pnpm build`, `pnpm lint`, `pnpm typecheck` pass from clean clone |
| P0.2 | `packages/config`: Zod env schema; apps fail fast on missing env | Missing var → process exits with a clear message; `.env.example` lists all vars from [tech-stack.md](tech-stack.md) |
| P0.3 | `packages/db`: Drizzle setup, migration runner, Testcontainers test helper, seed script | `pnpm db:migrate` and `pnpm db:seed` work locally (docker compose Postgres) |
| P0.4 | Fastify skeleton: health check, error format, request IDs, pino with redaction, Sentry | `GET /v1/health` → 200; error shape matches [api-spec.md](api-spec.md); a phone number in a log is redacted (test) |
| P0.5 | Worker skeleton with pg-boss, one sample cron | Job runs locally; failures retry with backoff |
| P0.6 | Next.js skeleton with Tailwind + shadcn, route groups `(owner) (clinical) (admin) (counter)`, API client | Pages render placeholder layouts |
| P0.7 | CI (GitHub Actions): lint, typecheck, unit + integration tests, migration drift check, gitleaks | Green on main; PRs blocked on failure |
| P0.8 | Render deploy (staging): api, worker, web, Postgres | Staging URLs live; migrations run on deploy |

## Phase 1: Tenancy, auth, onboarding basics
Goal: an admin can create a verified pharmacy, its owner can log in, and a tablet can pair.

| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P1.1 | Schema: pharmacies, staff, staff_roles, devices, device_pairing_codes, events | — | Migrations match [data-model.md](data-model.md) |
| P1.2 | Tenant context + RLS + `requireRole` / `requireDevice` guards | — | Cross-tenant test suite: requests with another pharmacy's IDs → 404 |
| P1.3 | Admin Google OAuth + allow-list | AD-1 | Non-allow-listed email rejected |
| P1.4 | Admin: create pharmacy, staff, verify, generate join code | AD-2 | Pharmacy can't go `verified` without licence + PCN no.; join code unique, format `AAA-9999` |
| P1.5 | Staff OTP login (WhatsApp auth template + Termii fallback), sessions, pharmacy switcher | — | OTP TTL 5 min, 5 attempts, rate limit; both roles work for one person |
| P1.6 | Device pairing (code → token), list/revoke | CP-4 | Code single-use, 10 min; revoked token → 401 immediately |
| P1.7 | QR poster PDF (`wa.me/{REXA}?text=Hi%20Rexa!%20JOIN%20{code}`) | CJ-1 | Scanning the poster opens WhatsApp with the pre-filled text |

## Phase 2: Rexa enrolment
Goal: a real person scans a QR and completes enrolment on WhatsApp.

| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P2.1 | Schema: customers, consents, conversations, messages, medicines, medication_plans, refill_cycles | — | |
| P2.2 | WhatsApp webhook: verify signature, dedupe, persist, enqueue; status updates | — | Bad signature → 401; duplicate `wa_message_id` processed once |
| P2.3 | `packages/whatsapp`: send text/interactive/template, 24h-window guard, template registry | — | Free-form outside 24h throws; retries on 5xx; permanent errors marked failed |
| P2.4 | `packages/rexa` engine: pure `step()` + state machine for new → consent → capture → duration → enrolled | CJ-1..5 | Table-driven tests cover every transition in [conversation-design.md](conversation-design.md) §2, including STOP/HELP in every state |
| P2.5 | Consent recording (versioned, scoped, with evidence) | CJ-2 | No medication data stored before consent (test) |
| P2.6 | `packages/ai`: `extractMedicine` (Claude vision, forced tool use, Zod-validated) + `parseDuration` | CJ-3, CJ-4 | Golden set ≥ 50 fixtures; ≥ 90% name accuracy on recorded responses; low confidence → typed-input fallback |
| P2.7 | Text capture path (typed medicine) | CJ-3 | "amlodipine 5mg 30 tabs" → correct read-back |
| P2.8 | Enrolment completion: create medicine (if new), plan, first cycle; assign experiment arm | CJ-5, MB-1 | Arm assigned once per customer; split within ±5% of `control_share` over 10k simulated |
| P2.9 | Fallbacks + handoff flagging | CJ-7 | Two unknowns → handoff offer; flagged conversation visible via API |
| P2.10 | Photo storage to R2 with 30-day lifecycle | — | Object key on plan; lifecycle rule configured |

**Demo:** scan poster → consent → photo of an Amlodipine pack → "1 month" → "All set!"

## Phase 3: Reminders and refill cycles
Goal: reminders go out on time to the right arm, and Yes/No replies are handled.

| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P3.1 | `domain.startCycle`, `cycleStatus`, reminder time calculation (Lagos tz, 09:00–19:00 window) | CJ-6 | Unit tests incl. month ends, DST-free tz, leap year |
| P3.2 | scheduled_messages + `reminders.dispatch` cron (SKIP LOCKED, idempotent) | CJ-6 | Two concurrent workers never double-send (test) |
| P3.3 | Templates `refill_reminder_v1` / `_late_v1` submitted and approved | CJ-6 | Approved status in registry |
| P3.4 | Reminder reply handling: Yes → refill_requested; No → declined | CJ-6 | Reply to an old reminder maps to the correct cycle (via context message ID) |
| P3.5 | Guards: no send if consent withdrawn, plan paused/stopped, clinical hold, or vetoed | OW-5 | Each guard has a test |

## Phase 4: Counter Panel
Goal: an attendant can run their day from the tablet.

| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P4.1 | `/counter` PWA shell (manifest, install prompt, large-tap layout), pairing screen | CP-4 | Installable on Android Chrome; Lighthouse PWA pass |
| P4.2 | Due list: Due (2 days) / Overdue (≤ 7 days) tabs, "Requested" badge, search | CP-1 | Matches `cycleStatus`; masked phones |
| P4.3 | Record action: Collected refill / Didn't have it, confirm + 30s undo | CP-2 | Collected → next cycle created + pending reminder cancelled; Didn't have it → stock_request |
| P4.4 | Offline queue (TanStack Query persisted mutations, `client_action_id`) | CP-5 | Airplane mode → actions queue → sync on reconnect, no duplicates |
| P4.5 | Stock check: search / barcode scan, set status, recent checks with "ago" | CP-3 | Barcode scan fills medicine; recent list ordered newest first |

## Phase 5: Unavailable medicine and stock queries
Goal: the full "When a medicine is unavailable" flow works.

| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P5.1 | Intent classifier (`ai.classifyIntent`, Haiku) + keyword shortcuts | CI-1..4 | Labelled set ≥ 100 utterances incl. Pidgin phrasing; ≥ 90% accuracy |
| P5.2 | Stock query flow: fresh-check answer, else pending query → counter → reply | CI-1, CP-3 | End-to-end test: WhatsApp question → counter answer → customer reply |
| P5.3 | Pending queries on Counter Panel (poll 15s, timer) + 10-min timeout message | CP-3 | Timed-out queries flagged in workspace |
| P5.4 | Asked-for creation from WhatsApp "notify me", counter "Didn't have it", walk-ins | 5.4 | All three sources appear in one list |
| P5.5 | Request status intent | CI-2 | Correct status per open request |
| P5.6 | Now in stock → Notify → `back_in_stock_v1` to waiting customers | OW-4 | Only consented customers with `stock_alerts` scope notified; counts recorded |

## Phase 6: Owner Workspace
| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P6.1 | Workspace shell, nav, pharmacy switcher | — | |
| P6.2 | Dashboard: due soon / overdue / lapsed + returns card (rolling 30d, provisional) | OW-1 | Numbers match domain functions on seeded data |
| P6.3 | Customers lists (3 tabs), detail page, CSV export | OW-2 | Lapsed = > 7 days overdue |
| P6.4 | Asked for / Unavailable list, grouping, status changes | OW-3 | |
| P6.5 | Restock notifications page | OW-4 | |
| P6.6 | Flagged conversations | OW-8 | |
| P6.7 | Settings: profile, staff invites, devices, QR poster, catalogue + prices | OW-7 | |

## Phase 7: Clinical page
| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P7.1 | Patient search + history (access logged) | OW-5 | `clinical.viewed` event written |
| P7.2 | Clinical flags (alert, hold reminders, exclude from experiment) | OW-5 | Hold stops reminders; exclusion → arm `excluded` |
| P7.3 | Upcoming messages review + veto | OW-5 | Vetoed message never sent |
| P7.4 | Consent settings (scopes, notice URL, version) | OW-5 | New version doesn't alter existing consents |

## Phase 8: Other customer interactions
| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P8.1 | Medication history | CI-3 | |
| P8.2 | Update plan / stop medicine / add medicine | CI-4 | Update recalculates the open cycle's due date and reschedules the reminder |
| P8.3 | Change consent scopes + STOP withdrawal + erasure job | CI-4 | Withdrawal cancels all pending sends; erasure leaves only anonymised cycles/events |
| P8.4 | Multi-pharmacy disambiguation | ADR-002 | Customer at 2 pharmacies is asked which one |

## Phase 9: Measurement & billing
| ID | Task | Req | Acceptance criteria |
|---|---|---|---|
| P9.1 | `domain.metrics`: eligibility, returns, lift + Newcombe CI, pooled fallback | MB-2, MB-3 | Golden test reproduces the worked example exactly |
| P9.2 | `measurement.recompute` nightly + snapshot finalisation | MB-4 | Idempotent; final snapshots immutable |
| P9.3 | Reports page + PDF | OW-6 | Matches [measurement-and-billing.md](measurement-and-billing.md) §6 |
| P9.4 | Pricing plans + invoice generation (draft) | MB-4 | All three plan kinds tested; negative lift → 0 performance fee |
| P9.5 | Admin issue flow + Paystack payment link + webhook | MB-4 | Paid webhook marks invoice paid; signature verified |

## Phase 10: Pilot hardening and launch
| ID | Task | Acceptance criteria |
|---|---|---|
| P10.1 | Admin console: pharmacy health, failed messages + retry, audit log, template status | AD-1 complete |
| P10.2 | Retention jobs (photos 30d, payloads 90d) | Verified on staging |
| P10.3 | Security review: cross-tenant tests, webhook forgery, rate limits, dependency audit | No high findings open |
| P10.4 | Load test: 50 pharmacies × 2k customers; reminder burst of 10k at 10:00 | Dispatch finishes < 15 min; API p95 within NFRs |
| P10.5 | Onboarding kit: training guide for attendants, poster, DPA template | Signed off by founders |
| P10.6 | Pilot launch with 3–5 pharmacies; weekly metrics review | Success metrics tracked ([prd.md](prd.md) §7) |

## Dependency graph (phases)

```
P0 → P1 → P2 → P3 ─┬→ P4 → P5 ─┐
                   │           ├→ P6 → P7 → P9 → P10
                   └→ P8 ──────┘
```
P4 and P8 can run in parallel after P3. P9.1 (pure metrics) can start any time after P2.
