# Progress

Live tracker for [implementation-plan.md](implementation-plan.md). **Update this file in the same PR as the work.**

Legend: `[ ]` todo · `[~]` in progress · `[x]` done · `[!]` blocked (say why in the log)

**Current phase:** Phase 0: Foundations
**Next task:** P0.1

## Phase 0: Foundations
- [ ] P0.1 Scaffold monorepo
- [ ] P0.2 Env config (Zod)
- [ ] P0.3 DB package, migrations, Testcontainers
- [ ] P0.4 Fastify skeleton, logging, Sentry
- [ ] P0.5 Worker skeleton (pg-boss)
- [ ] P0.6 Next.js skeleton + route groups
- [ ] P0.7 CI pipeline
- [ ] P0.8 Staging deploy on Render

## Phase 1: Tenancy, auth, onboarding basics
- [ ] P1.1 Schema: pharmacies, staff, devices, events
- [ ] P1.2 Tenant context, RLS, guards
- [ ] P1.3 Admin OAuth
- [ ] P1.4 Admin pharmacy onboarding + verify + join code
- [ ] P1.5 Staff OTP login
- [ ] P1.6 Device pairing
- [ ] P1.7 QR poster PDF

## Phase 2: Rexa enrolment
- [ ] P2.1 Schema: customers → refill_cycles
- [ ] P2.2 WhatsApp webhook
- [ ] P2.3 WhatsApp client + template registry
- [ ] P2.4 Rexa engine (enrolment states)
- [ ] P2.5 Consent recording
- [ ] P2.6 AI medicine extraction + duration parsing
- [ ] P2.7 Typed medicine capture
- [ ] P2.8 Enrolment completion + arm assignment
- [ ] P2.9 Fallbacks + handoff
- [ ] P2.10 Photo storage (R2, 30-day lifecycle)

## Phase 3: Reminders and refill cycles
- [ ] P3.1 Cycle domain logic
- [ ] P3.2 Reminder dispatcher
- [ ] P3.3 Reminder templates approved
- [ ] P3.4 Yes/No reply handling
- [ ] P3.5 Send guards (consent, hold, veto)

## Phase 4: Counter Panel
- [ ] P4.1 PWA shell + pairing screen
- [ ] P4.2 Due list
- [ ] P4.3 Record action + undo
- [ ] P4.4 Offline queue
- [ ] P4.5 Stock check + barcode

## Phase 5: Unavailable medicine and stock queries
- [ ] P5.1 Intent classifier
- [ ] P5.2 Stock query flow
- [ ] P5.3 Pending queries + timeout
- [ ] P5.4 Asked-for creation (all sources)
- [ ] P5.5 Request status intent
- [ ] P5.6 Now in stock → notify

## Phase 6: Owner Workspace
- [ ] P6.1 Shell + nav
- [ ] P6.2 Dashboard
- [ ] P6.3 Customers lists + detail + export
- [ ] P6.4 Asked for list
- [ ] P6.5 Restock notifications
- [ ] P6.6 Flagged conversations
- [ ] P6.7 Settings

## Phase 7: Clinical page
- [ ] P7.1 Patient history
- [ ] P7.2 Clinical flags
- [ ] P7.3 Message review + veto
- [ ] P7.4 Consent settings

## Phase 8: Other customer interactions
- [ ] P8.1 Medication history
- [ ] P8.2 Update / stop / add medicine
- [ ] P8.3 Consent change, STOP, erasure
- [ ] P8.4 Multi-pharmacy disambiguation

## Phase 9: Measurement & billing
- [ ] P9.1 Metrics domain
- [ ] P9.2 Nightly recompute + finalisation
- [ ] P9.3 Reports page + PDF
- [ ] P9.4 Pricing + invoice drafts
- [ ] P9.5 Issue + Paystack

## Phase 10: Pilot hardening and launch
- [ ] P10.1 Admin console complete
- [ ] P10.2 Retention jobs
- [ ] P10.3 Security review
- [ ] P10.4 Load test
- [ ] P10.5 Onboarding kit
- [ ] P10.6 Pilot launch

---

## Open decisions blocking work
See [decisions.md](decisions.md). Move a row here when it actually blocks a task.

| Decision | Blocks | Owner | Status |
|---|---|---|---|
| — | — | — | — |

## Log
Newest first. One line per meaningful change: `YYYY-MM-DD · task · what happened · PR/commit`.

- 2026-10-08 · — · Added docs/cost-estimate.md (service pricing + run cost by stage) · —
- 2026-10-05 · — · Spec set created (PRD, architecture, tech stack, data model, API, conversation design, measurement & billing, security, plan, rules) · —
