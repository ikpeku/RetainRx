# Decisions Log (ADRs) & Open Questions

Record every significant decision here. Format: context → decision → consequences. Never rewrite an accepted ADR; supersede it with a new one.

## Accepted

### ADR-001: TypeScript monorepo, Postgres as the only stateful store
- **Context:** Small team, three surfaces plus a worker, pilot budget.
- **Decision:** pnpm + Turborepo monorepo; Fastify API, Next.js web, Node worker; PostgreSQL for data **and** the job queue (pg-boss). No Redis.
- **Consequences:** One database to back up and secure; jobs and domain writes can share a transaction. If queue throughput ever becomes a bottleneck (unlikely below ~1M messages/month), revisit.

### ADR-002: One shared Rexa WhatsApp number for all pharmacies
- **Context:** Each pharmacy getting its own WhatsApp Business number means per-pharmacy Meta verification and template approval, which slows onboarding.
- **Decision:** One RetainRx-owned number. The pharmacy is identified by the join code in the QR. Every message names the pharmacy. Customers are scoped per pharmacy.
- **Consequences:** Fast onboarding. A customer enrolled at two pharmacies needs disambiguation (Rexa asks "Which pharmacy?"). Moving to per-pharmacy numbers (white-label) later is a separate ADR.

### ADR-003: Counter Panel is a PWA route in the web app, authenticated by a paired device token
- **Context:** The design says "No login needed (device paired once)". Attendants change often and share the tablet.
- **Decision:** Owner generates a pairing code; tablet exchanges it for a revocable device token. Device tokens get a restricted, read-minimal scope.
- **Consequences:** Anyone holding the tablet can act. This is mitigated by limited scope, masked phones, owner revocation and `last_seen_at`.

### ADR-004: Status buckets are derived, not stored
- **Decision:** `due / overdue / lapsed` are computed from `due_date` and the Lagos date at read time.
- **Consequences:** No nightly status-flip job and no drift. Queries filter on `due_date` ranges (indexed).

### ADR-005: Delayed-reminder control group, disclosed in the privacy notice
- **Context:** Proving impact needs a counterfactual. Withholding reminders entirely is ethically weaker.
- **Decision:** Control customers get the same reminder 7 days later. The default split is 80/20. Disclosed in the privacy notice. The superintendent can exempt clinically high-risk patients (excluded from measurement).
- **Consequences:** Smaller measured lift than a no-reminder control would show (conservative). It needs enough volume per pharmacy, so a network-pooled fallback exists.

### ADR-006: Measurement window anchored on the on-time reminder for both arms
- **Decision:** Return = collected within `[due−3d, due+4d)` at reminder time, for both arms. See [measurement-and-billing.md](measurement-and-billing.md).
- **Consequences:** A clean "reminded vs not yet reminded" contrast. The control's own reminder falls outside the window by design.

### ADR-007: AI only proposes, the customer confirms
- **Decision:** Claude extracts medicine details and classifies intents through schema-forced tool use. Nothing is persisted as a plan without explicit customer confirmation. AI never triggers sends or state changes directly.
- **Consequences:** Robust to hallucination and prompt injection, at the cost of one extra confirmation step in enrolment.

## Open questions (need a decision before the noted phase)

| ID | Question | Needed by | Proposed default |
|---|---|---|---|
| D-1 | Pricing: flat, performance (% of incremental revenue) or hybrid? What values? | Phase 9 | Pilot: ₦0 flat with reports. After pilot: hybrid (small flat fee + % of incremental revenue, capped) |
| D-2 | Data residency: is EU hosting (Render Frankfurt) acceptable under the NDPA, or do we need in-country/African hosting? ⚖️ | Before pilot launch | EU + DPA + transfer safeguards; revisit for scale |
| D-3 | "No" on a reminder: should Rexa ask why (still have supply / bought elsewhere / stopped)? | Phase 3 | MVP: no reason asked; v1.1: optional quick-reply reason |
| D-4 | Should the control share be 80/20 or 50/50 during the pilot for faster statistical power? | Phase 3 | 70/30 for the pilot only, then 80/20 |
| D-5 | Walk-in "Do you have X?" at the counter: capture phone for notification, with verbal consent? | Phase 5 | Optional phone; the customer gets a WhatsApp consent message before any notification |
| D-6 | Owner is also an attendant: allow the owner session to use the Counter Panel without pairing? | Phase 4 | Yes, owner session can open `/counter` |
| D-7 | Languages: which comes after English? | Post-MVP | Nigerian Pidgin |
| D-8 | Refill value: do pharmacies enter prices per medicine, or only a default average? | Phase 9 | Default average at onboarding; per-medicine optional |
