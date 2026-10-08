# Rexa: Conversation Design

Rexa is the pharmacy's WhatsApp assistant. This file defines its voice, state machine, intents, message copy and WhatsApp templates. Copy lives in `packages/rexa/copy/en.ts` and must match this file.

## 1. Voice and rules

- Warm, brief, plain English. Short sentences. At most one question per message.
- Always speaks **for the pharmacy**: "your pharmacy assistant at Ada Pharmacy".
- **Never gives medical advice** (dosing, substitutions, side effects, interactions). Response: "That's a question for the pharmacist. I've let Ada Pharmacy know, and they'll reach out." The conversation is then flagged.
- Always reads back what it understood before saving anything.
- Use buttons wherever possible (interactive reply buttons, max 3; lists for more).
- `STOP`, `UNSUBSCRIBE` or `OPT OUT` (any case) **always** works, in any state, and goes to the withdrawal confirmation.
- `HELP` or `MENU` in any state shows the menu.

## 2. State machine

```
                 ┌──────────────┐  join code
 (first msg) ───▶│  new         │────────────▶ consent.awaiting
                 └──────────────┘  no code → "Please scan the QR at your pharmacy" (stays new)

consent.awaiting ──[Confirm my details]──▶ consent.awaiting_name? ──▶ capture.awaiting_medicine
                 ──[No thanks]──────────▶ declined (no data kept beyond the message log)

capture.awaiting_medicine ──photo──▶ (AI extract) ──conf≥0.8──▶ capture.confirm_medicine
                          ──text───▶ (AI parse)   ──conf<0.8──▶ capture.awaiting_medicine (ask to type; retry≤2 → handoff)
capture.confirm_medicine ──[Yes, that's right]──▶ capture.awaiting_duration
                         ──[Fix it]──────────────▶ capture.awaiting_medicine
capture.awaiting_duration ──[2 weeks|1 month|2 months|text]──▶ enrolled (plan + cycle created)
enrolled ──[Add another medicine]──▶ capture.awaiting_medicine
enrolled ──[Done]──▶ idle

idle ──intent──▶ { stock_query | request_status | history | update | refill_reply | menu | handoff | stop }
reminder sent ──[Yes]──▶ refill requested ──▶ idle
              ──[No]───▶ cycle declined   ──▶ idle
```

The engine is `rexa.step(state, context, input) → { nextState, context, outbound[], commands[] }`, a pure function. `commands` are domain operations the worker executes (e.g. `CreatePlan`, `RecordConsent`, `CreateStockQuery`).

## 3. Enrolment copy

**Pre-filled QR text:** `Hi Rexa! JOIN {JOIN_CODE}`

**consent.awaiting** (interactive buttons):
> Hi! I'm Rexa, your pharmacy assistant at **{pharmacy}**. I'll remind you when your medicine is due and help with quick questions.
>
> To do this, {pharmacy} will keep your name, phone number and the medicines you tell me about. You can stop at any time by replying STOP. Privacy notice: {privacy_url}
>
> To get started, please confirm your name: **{wa_profile_name}**
>
> [Confirm my details] [Change name] [No thanks]

**capture.awaiting_medicine:**
> Can you tell me what medicines you bought last? You can send a photo of the pack or type it.

**capture.confirm_medicine:**
> Got it!
> **{medicine} {strength}**
> • {quantity} {unit}
>
> [Yes, that's right] [Fix it]

(If quantity is missing: "How many {unit} did you buy?")

**capture.awaiting_duration:**
> How long does this supply last? (e.g. 1 month, 2 months)
> [2 weeks] [1 month] [2 months]

Duration parsing: "1 month" = 30 days, "2 months" = 60, "x weeks" = 7x, "x days" = x. Allowed range 7–180 days; outside that, Rexa asks again.

**enrolled:**
> All set! I'll remind you 3 days before your supply runs out.
> ✅ {medicine} {strength} · {quantity} {unit} · {duration_label}
>
> [Add another medicine] [Done]

> Note: the copy says "3 days before" for both arms. Control-arm customers are informed in the privacy notice that reminder timing can vary while the pharmacy measures service quality. See ADR-005.

## 4. Reminder copy (template, outside the 24h window)

Template `refill_reminder_v1` (category: **UTILITY**, language `en`):
> Reminder from {{1}}: Your {{2}} is due in {{3}} days. Need more? Reply "Yes" to request a refill or "No" if you're all set.
> Quick replies: [Yes] [No]

- `{{3}}` = days until the due date; for the control arm this becomes "was due {{n}} days ago", using template `refill_reminder_late_v1`:
> Reminder from {{1}}: Your {{2}} was due {{3}} days ago. Need more? Reply "Yes" to request a refill or "No" if you're all set.

**On "Yes":**
> Great, I've told {pharmacy} to get your {medicine} ready. 👍

The cycle gets `refill_requested_at` and the counter due list shows a "Requested" badge.

**On "No":**
> No problem. I'll check in again next time. Reply MENU anytime.

The cycle outcome becomes `declined`. The plan stays active; the next cycle starts at the due date.

## 5. Intents (after enrolment)

Classified by `ai.classifyIntent` (Haiku, tool-use schema) only when the input is free text and not a button reply. Keyword shortcuts run first (STOP, HELP, MENU, YES, NO).

| Intent | Examples | Handler |
|---|---|---|
| `stock_query` | "Do you have Ozempic?", "una get Lisinopril?" | Fresh check (≤ freshness window) → answer; else create stock_query → "Let me check with {pharmacy}, I'll reply in a few minutes." |
| `request_status` | "Is it in stock yet?", "any update on the Lisinopril?" | Look up the customer's open stock_requests → status reply |
| `history` | "What do I have on record?" | List active plans with next due dates and the last 3 collections |
| `update_plan` | "I now take 10mg", "change to 2 months" | Confirm which plan → update → read back |
| `stop_medicine` | "I stopped taking Metformin" | Confirm → plan stopped |
| `update_details` | "Change my name" | Name update flow |
| `change_consent` | "Change what I agree to" | List of scopes as toggles (reminders / stock alerts) + "Stop everything" |
| `add_medicine` | "Add another medicine" | Go to capture.awaiting_medicine |
| `medical_question` | "Can I take this with alcohol?" | Refuse advice + flag for pharmacist |
| `handoff` | "I want to talk to someone" | Flag conversation + "{pharmacy} will reach out" + wa.me link to the pharmacy if they have one |
| `unknown` | anything else | Menu; second consecutive unknown → offer handoff |

**Menu** (list message):
> What can I help with?
> • Ask if something is in stock
> • Check on something I asked for
> • See my medicines
> • Add a medicine
> • Update my details
> • Talk to the pharmacy

## 6. Stock answers

| Situation | Copy |
|---|---|
| Fresh check, in stock | "Yes! {medicine} is in stock at {pharmacy} (checked {ago})." |
| Fresh check, low | "{pharmacy} has a little {medicine} left (checked {ago}). Want me to ask them to keep one for you? [Yes, please] [No]" |
| Out / Didn't have it | "Sorry, {pharmacy} doesn't have {medicine} right now. Want me to tell you when it arrives? [Yes, notify me] [No]" → `stock_request` |
| Pending check | "Let me check with {pharmacy}. I'll reply in a few minutes." |
| Timed out (10 min) | "{pharmacy} is busy right now. I'll message you as soon as they check." |

Template `back_in_stock_v1` (UTILITY):
> Good news from {{1}}: {{2}} is now in stock. Reply "Yes" if you'd like them to keep one for you.

## 7. Withdrawal (STOP)

> Are you sure you want to stop all messages from {pharmacy}? Your data will be deleted within 30 days.
> [Yes, stop] [No, keep me]

On confirm: consent `withdrawn`, all pending scheduled messages cancelled, customer status `withdrawn`, erasure job scheduled.
> Done. You won't hear from me again. If you change your mind, just scan the QR at {pharmacy}.

## 8. AI extraction contract

`ai.extractMedicine(image | text) → { medicines: [{ name, strength?, form?, quantity?, quantity_unit?, confidence }], notes? }`

- Implemented with Claude tool use (forced tool call) so the output is always schema-valid, then validated with Zod.
- The prompt instructs: extract only what is visible or stated. Never guess the strength. Return confidence per field.
- Name normalised to a generic name where the brand is known, keeping the brand in `notes`.
- Confidence < 0.8, or more than one medicine detected in a photo, makes Rexa confirm each one separately.
- Photos are stored in R2, only the R2 key is kept on the plan, and they are deleted after 30 days.
- A golden test set of ≥ 50 labelled pack photos and text strings (in `packages/ai/fixtures`) runs in CI against recorded responses. Live evaluation runs on demand.

## 9. WhatsApp templates registry

| Name | Category | Used for |
|---|---|---|
| `refill_reminder_v1` | UTILITY | On-time reminder |
| `refill_reminder_late_v1` | UTILITY | Control-arm reminder |
| `back_in_stock_v1` | UTILITY | Restock notification |
| `stock_query_answer_v1` | UTILITY | Answering a stock query when the 24h window has closed |
| `staff_otp_v1` | AUTHENTICATION | Owner/superintendent login code |

Templates are versioned. Changing copy means a new version, Meta approval, and then a switch in the registry (`packages/whatsapp/templates.ts`).
