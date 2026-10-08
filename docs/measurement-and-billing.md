# Measurement & Billing

This is how RetainRx proves, **in naira**, that it worked, and how a pharmacy is invoiced. The logic lives in `packages/domain/metrics.ts` and `packages/domain/pricing.ts`. It is pure and fully unit-tested.

## 1. Experiment design

- **Unit of randomisation:** the customer (per pharmacy), assigned at first enrolment.
- **Arms:**
  - `on_time`: reminder at **due_date − 3 days**, at the pharmacy's reminder time (default 10:00 WAT).
  - `control`: the same reminder **7 days later**, at **due_date + 4 days**.
  - `excluded`: clinically exempted by the superintendent. Always gets the on-time reminder and is **not** counted.
- **Allocation:** `control` with probability `pharmacy.control_share` (default 0.20), otherwise `on_time`. Uses a seeded CSPRNG, and the assignment is logged to `events` (`experiment.assigned`).
- **Immutability:** the arm never changes, except to `excluded`. Each `refill_cycle` freezes the arm at creation.
- **Ethics:** everyone is reminded; the control is only delayed. This is disclosed in the privacy notice (ADR-005).

## 2. Outcome definition

For every refill cycle `c` with `ontime_reminder_at = due_date − 3d @ reminder_time`:

```
window(c)  = [ontime_reminder_at, ontime_reminder_at + 7 days)
return(c)  = c.collected_at ∈ window(c)
```

The window is **the same calendar window for both arms**: the 7 days after an on-time reminder is (or would have been) sent. It ends at due_date + 4d, exactly when the control reminder fires, so the comparison is "reminded on time" vs "not yet reminded". This is a clean intent-to-treat contrast and matches the dashboard label **"Returns (7 days after reminder)"**.

**Eligible cycles for a period** (month M):
- `ontime_reminder_at` falls in month M (Lagos time)
- the window has fully elapsed (for `final` snapshots)
- the customer arm is `on_time` or `control` (not `excluded`)
- the cycle was not ended by withdrawal or a clinical hold before the window started

Outcomes `declined` and `unavailable` count as **non-returns**. "Didn't have it" is pharmacy-side lost demand, and it is reported separately so the owner sees it.

## 3. Metrics

```
n_t, r_t  = eligible on-time cycles, returns among them
n_c, r_c  = eligible control cycles, returns among them
p_t = r_t / n_t            # e.g. 68%
p_c = r_c / n_c            # e.g. 42%
lift = p_t − p_c           # e.g. 26 percentage points
lift CI 95% = Newcombe hybrid score interval (difference of two proportions)

incremental_returns  = lift × n_t
avg_refill_value     = mean refill_value_kobo over on-time returns in the period
                       (fallback: pharmacy.default_refill_value_kobo)
incremental_revenue  = incremental_returns × avg_refill_value      (kobo, rounded half-even)
```

Negative lift is reported as is. Billing treats a negative lift as 0 (see §5).

### Small samples

If `n_c < 30` or `n_t < 30` for the pharmacy-month:
- `lift_source = network_pooled`: use the pooled lift across all active pharmacies for that month (cycle-weighted), still applied to *this* pharmacy's `n_t` and refill value.
- The report says so plainly: "Not enough reminders yet to measure your pharmacy alone; we used the average across RetainRx pharmacies."

### Worked example (from the design)

| | On-time | Control |
|---|---|---|
| Eligible cycles | 250 | 60 |
| Returns | 170 | 25 |
| Rate | 68% | 42% |

lift = 0.26 → incremental returns = 0.26 × 250 = **65** → at ₦12,500 avg refill → **₦812,500 incremental revenue** that month.

## 4. Snapshots and timing

- A nightly job at 02:00 WAT (`measurement.recompute`) recomputes `measurement_snapshots` for the current and previous month (`status=provisional`).
- A month becomes `final` when every eligible cycle's window has closed: on day 8 of the next month at 02:00 WAT, plus a safety margin of 1 day, so day 9.
- Dashboard (OW-1) shows the **rolling last 30 days** (provisional) and the last final month.

## 5. Pricing and invoicing

Pricing plans (`pricing_plans.kind`):

| Kind | Formula |
|---|---|
| `flat` | `flat_fee_kobo` |
| `performance` | `clamp(performance_pct × max(incremental_revenue, 0), min_fee, max_fee)` |
| `hybrid` | `flat_fee_kobo + performance_pct × max(incremental_revenue, 0)`, capped at `max_fee` |

> **Open decision (D-1 in [decisions.md](decisions.md)):** the default plan and its values. The schema supports all three. Pilot pharmacies get a `flat` plan of ₦0 (free pilot), and reports are still generated.

Invoice flow:
1. On day 9 at 06:00 WAT, `billing.generate_invoices` creates a `draft` invoice per active pharmacy from the final snapshot. Line items: plan fee, measured lift, incremental returns, incremental revenue, and the computation.
2. An admin reviews drafts in the admin console and clicks **Issue** (MVP; auto-issue later).
3. Issuing creates a Paystack payment request, renders the PDF (report + invoice) to R2, and messages the owner on WhatsApp with the link.
4. The Paystack webhook (`charge.success`) marks the invoice `paid`.
5. Invoice numbers are sequential per year: `RRX-YYYY-NNNNNN`.

## 6. Report contents (PDF + workspace page)

1. Headline: "RetainRx brought back **65 extra refills** worth **₦812,500** in September."
2. Return rate bars: on-time vs 7-day-late, with the CI.
3. Funnel: enrolled → reminded → requested refill → collected.
4. Lost demand: "Didn't have it" count and top 5 asked-for medicines.
5. Due / overdue / lapsed counts at month end.
6. Method note (plain language) + lift source.

## 7. Guardrails

- Metrics code is pure and deterministic, with golden tests for the worked example and edge cases (zero controls, all returns, negative lift).
- Arm assignment cannot be edited via the API or the admin console.
- Collections recorded more than 30 days after the fact (backdated) are rejected by the API, so the measurement can't be gamed.
- Every invoice stores its `snapshot_id`, so an invoice is always reproducible.
