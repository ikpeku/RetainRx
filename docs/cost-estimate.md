# Cost Estimate: Third-Party Services

What it costs RetainRx to run the stack in [tech-stack.md](tech-stack.md). Prices were checked on 2026-10-08. Amounts are in USD because most vendors bill in dollars. Naira figures use **₦1,400 / $1**: the CBN official rate is about ₦1,329 and the parallel rate about ₦1,390, and the extra margin covers the FX spread on dollar cards.

> Re-check the  rows before you budget. They are either secondary-source figures or pricing that is changing.

## 1. Unit prices

| Service | What we use it for | Price | Source |
|---|---|---|---|
| **WhatsApp Cloud API**: utility template  | Reminders, back-in-stock alerts, late stock answers | **~$0.0101 per delivered message** (Nigeria). Lower rates unlock with monthly volume | Meta rate card (CSV) — Nigeria figure from a secondary source |
| WhatsApp: authentication template | Staff OTP login | About the utility rate or lower | Meta rate card |
| WhatsApp: service messages (free-form replies inside the 24h window)  | All Rexa conversation replies | **Free** according to Meta's docs. One secondary source says they cost about $0.0101 from 1 Oct 2026 | See §5 |
| **Claude Sonnet 5.5** (`claude-sonnet-5-5`) | Medicine photo extraction | $2 / M input tokens, $10 / M output tokens; cache reads $0.20 / M | Anthropic |
| **Claude Haiku 4.5** (`claude-haiku-4-5`), current spec | Intent classification | $1 / M input, $5 / M output | Anthropic |
| Claude Haiku 5.5 (`claude-haiku-5-5`), alternative | Intent classification | $0.10 / M input, $0.50 / M output (10× cheaper) | Anthropic |
| **Render** | Hosting | Web service: Starter $7, Standard $25, Pro $85 per month. Background worker from $7 per month. Postgres: Basic $20, Standard $95, Pro $185 per month. Workspace seats are extra (roughly $19 per user per month on Professional ) | Render |
| **Cloudflare R2** | Photos, PDFs | $0.015 per GB-month, $4.50 per M writes, $0.36 per M reads, **egress free**. Free tier: 10 GB, 1M writes, 10M reads per month | Cloudflare |
| **Sentry** | Error tracking | Developer free (1 user, 5k errors); Team $26 per month; Business $80 per month | Sentry |
| **Termii**  | SMS fallback for OTP | About ₦6 per SMS (Nigerian retail rate; Termii's own rate card not confirmed) | Market rate |
| **Paystack** | Collecting invoices from pharmacies | 1.5% + ₦100 per payment, **capped at ₦2,000**; the ₦100 is waived under ₦2,500 | Paystack |
| Other | Domain, GitHub, Google Workspace (admin login) | About $15 per year; GitHub free or Team $4 per user; Workspace about $7 per user | — |

## 2. Usage assumptions per customer

| Driver | Assumption |
|---|---|
| Active medicines per customer | 1.3, on mostly monthly cycles |
| Template messages per customer per month | **1.4** (1.3 reminders + 0.1 back-in-stock) |
| Service messages per customer per month | **2** (reminder acknowledgements, stock answers, history) |
| Enrolment (one-off) | About 8 outbound service messages and 1.3 photo extractions |
| Photo extraction (Sonnet 5.5, effort `low`) | About 3,000 input tokens (image about 1,300 + prompt) and about 500 output tokens = **about $0.011 per photo** (about $0.015 per enrolment) |
| Intent classification | About 1 call per customer per month, about 800 tokens in and 100 out = $0.0013 (Haiku 4.5) or $0.00013 (Haiku 5.5) |
| Photo storage | About 1 MB per photo, deleted after 30 days |

## 3. Monthly cost by stage

Two figures are given for each stage: **A** = service messages stay free (Meta's current docs); **B** = service messages are charged from Oct 2026.

### Pilot: 5 pharmacies, 1,500 customers

| Line | $/month |
|---|---|
| WhatsApp templates (1,500 × 1.4 × $0.0101) | 21 |
| Claude: classification (Haiku 4.5) | 2 |
| Render: production (3 × Starter + Postgres Basic) | 41 |
| Render: staging (3 × Starter + Postgres Starter) | 28 |
| Render: workspace (2 seats) | 38 |
| R2 · Sentry · Termii · domain | ~3 |
| **Total A** | **~$133 (≈ ₦186k)** |
| WhatsApp service messages (1,500 × 2 × $0.0101) | +30 |
| **Total B** | **~$163 (≈ ₦228k)** |

One-off enrolment cost for 1,500 customers: about $23 of Claude, plus about $121 of WhatsApp under scenario B.

### Growth: 50 pharmacies, 20,000 customers, 2,000 new customers a month

| Line | $/month |
|---|---|
| WhatsApp templates | 283 |
| Claude: enrolment photos (2,000 × $0.015) | 30 |
| Claude: classification (Haiku 4.5) | 26 |
| Render (Standard api/web/worker, Postgres Standard, staging, 3 seats) | ~270 |
| Sentry Team | 26 |
| Paystack (50 invoices, about ₦1,500 each) | ~54 |
| R2 · Termii | ~6 |
| **Total A** | **~$695 (≈ ₦973k) → about $14 (≈ ₦19.5k) per pharmacy** |
| WhatsApp service messages + enrolment messages | +566 |
| **Total B** | **~$1,261 (≈ ₦1.77M) → about $25 (≈ ₦35k) per pharmacy** |

### Scale: 300 pharmacies, 150,000 customers, 15,000 new customers a month

| Line | $/month |
|---|---|
| WhatsApp templates (before volume discounts) | 2,121 |
| Claude: enrolment photos | 225 |
| Claude: classification (Haiku 4.5) | 195 |
| Render (2 × Pro api, Pro worker, Postgres Pro, staging, 4 seats) | ~600 |
| Sentry Business | 80 |
| Paystack (300 invoices at the ₦2,000 cap) | ~430 |
| R2 · Termii | ~25 |
| **Total A** | **~$3,676 (≈ ₦5.1M) → about $12 (≈ ₦17k) per pharmacy** |
| WhatsApp service messages + enrolment messages | +4,242 |
| **Total B** | **~$7,918 (≈ ₦11.1M) → about $26 (≈ ₦37k) per pharmacy** |

## 4. What drives the bill

1. **WhatsApp is 60–85% of variable cost.** Every reminder costs about ₦14. Volume tiers lower this at scale.
2. **Hosting is a fixed cost.** It dominates in the pilot and becomes small per pharmacy at scale.
3. **Claude is under 10%.** Photo extraction is a one-off per enrolment. Classification is negligible, more so on Haiku 5.5.
4. **Paystack** costs at most ₦2,000 per invoice. Decide whether RetainRx absorbs it or adds it to the invoice.

**Against revenue:** the worked example in [measurement-and-billing.md](measurement-and-billing.md) shows ₦812,500 of incremental revenue a month for one pharmacy. Running costs of ₦17k–37k per pharmacy are **2–5%** of that. Use this as a floor when setting the price in decision D-1.

## 5. Risks and levers

| Item | Action |
|---|---|
|  Service-message charging from Oct 2026 | Meta's pricing page doesn't list it, but a secondary source says it applies. **Confirm in WhatsApp Manager billing before the pilot.** If it applies, the WhatsApp bill roughly doubles |
| Enrolment is about 8 outbound messages | Combine messages (for example, put the read-back and the duration question in one message) to cut scenario-B cost and drop-off |
| Intent classification on Haiku 4.5 | Switching to `claude-haiku-5-5` is 10× cheaper and is the current Haiku. Run the P5.1 intent eval before switching (decision log) |
| Photo extraction prompt | Use prompt caching on the fixed system prompt and effort `low`; output stays short through tool use |
| FX exposure | Meta, Anthropic, Render and Sentry bill in USD. Keep a USD buffer and review the naira rate monthly |
| Render region | The EU region adds latency from Nigeria. Revisit with D-2 (data residency) |

## Sources

- Meta, WhatsApp Business Platform pricing: https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing
- Yournotify, WhatsApp pricing update Oct 2026: https://yournotify.com/blog/whatsapp-pricing-update-2026/
- Render pricing summaries: https://render.com/articles/how-much-does-cloud-application-hosting-cost-for-small-businesses · https://livemy.app/blog/render-pricing
- Cloudflare R2 pricing: https://developers.cloudflare.com/r2/pricing
- Sentry pricing: https://toolradar.com/tools/sentry/pricing
- Paystack fees (2026): https://kolonell.com/en/blog/paystack-vs-flutterwave-fees-comparison-nigeria-2026
- Nigeria SMS rates: https://www.sent.dm/resources/sms-pricing/nigeria-sms-pricing
- USD/NGN rate: https://techeconomy.ng/dollar-to-naira-naira-strengthens-to-132980-at-official-market-parallel-rate-at-1389
- Claude pricing: Anthropic model table (cached 2026-10-06)
