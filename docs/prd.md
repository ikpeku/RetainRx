# RetainRx — Product Requirements Document (PRD)

| | |
|---|---|
| Status | Draft v1 — source of truth for scope |
| Last updated | 2026-10-05 |
| Related | [architecture.md](architecture.md) · [conversation-design.md](conversation-design.md) · [measurement-and-billing.md](measurement-and-billing.md) · [data-model.md](data-model.md) |

---

## 1. Summary

RetainRx helps **independent pharmacies keep the people they already serve**, and **proves, in naira, that it worked**.

It is **one service with three touchpoints**:

1. **Rexa** — a WhatsApp assistant that enrols customers, remembers their medication, and reminds them when it is time to refill.
2. **Counter Panel** — a no-login tablet tab for the counter attendant with exactly three actions.
3. **Owner Workspace** — a web app where the pharmacy owner sees who is due back, who has lapsed, what people asked for that was unavailable, and the measured revenue impact.

These sit on a single backend that is the one source of truth. Founders and the support team run it through an admin console.

Product pillars (the six verbs):

| Pillar | Meaning | Where it happens |
|---|---|---|
| **Capture** | Customer shares what they bought, with consent | Rexa enrolment |
| **Remember** | We store the medication, dose, quantity and supply duration | Backend |
| **Remind** | Rexa messages the customer before they run out | Rexa reminders |
| **Reserve** | Customer says "Yes", and the pharmacy is told to prepare the refill. Unavailable items are logged and the customer is notified when they arrive | Rexa + Counter Panel + Owner Workspace |
| **Return** | Customer comes back and the attendant records the collection | Counter Panel |
| **Learn** | On-time vs 7-day-late control comparison gives incremental returns and revenue in ₦ | Measurement & Billing |

## 2. Problem

- Independent pharmacies in Nigeria lose chronic-medication customers (hypertension, diabetes, etc.) silently. The customer runs out, buys elsewhere or stops treatment, and the pharmacy never knows.
- Owners have no cheap way to know who is due back, who has lapsed, or what demand they failed to serve because they were out of stock.
- Retention tools that exist cannot **prove** they caused extra sales, so owners are reluctant to pay.
- Poor refill adherence is also a health problem for the patient.

## 3. Goals and non-goals

### Goals (MVP)
- G1. A customer can enrol in **under 2 minutes** on WhatsApp by scanning a QR code at the counter.
- G2. Enrolled customers get a reminder **3 days before** their supply runs out. A randomised control group gets the same reminder **7 days later**.
- G3. Attendants can see who is due, record a collection or "didn't have it", and answer stock questions **without logging in** and in **≤ 3 taps** per action.
- G4. Owners can see due, overdue and lapsed customers, unmet requests, and can notify customers when stock arrives.
- G5. Every month, each pharmacy gets a report of **incremental returns and incremental revenue in ₦**, with an invoice generated from it.
- G6. Explicit, recorded, revocable consent for every customer, and clinical oversight by the superintendent pharmacist.

### Non-goals (MVP)
- E-prescriptions, dispensing records, or replacing the pharmacy's POS or inventory system.
- Delivery or online payment by the customer.
- Full inventory management. Stock status is a lightweight "in stock / low / out" check, not a stock ledger.
- Clinical advice from Rexa. Rexa never gives dosing or medical advice. It hands off to the pharmacist.
- Languages other than English. Pidgin, Yoruba, Hausa and Igbo come after MVP.
- Native mobile apps. Everything is WhatsApp or web (PWA for the counter tablet).

## 4. Users and personas

| User | Touchpoint | Pays? | Auth | Core job |
|---|---|---|---|---|
| **Customer** (patient or caregiver) | Rexa on WhatsApp | No cost | WhatsApp phone number | Not run out of medicine; know if the pharmacy has something |
| **Counter attendant** | Counter Panel (tablet) | No cost | None. The device is paired once | Serve due customers, record outcomes, answer "do you have this?" |
| **Pharmacy owner** | Owner Workspace (web) | **Pays** | Phone + OTP | See who to win back, what demand was missed, and the ₦ impact |
| **Superintendent pharmacist** | Clinical page (web) | No cost | Phone + OTP | Clinical oversight: history, alerts, consent settings, veto or flag messages |
| **Founders & support** | Admin console | N/A | Staff SSO / OTP + allow-list | Onboard and verify pharmacies, support, troubleshooting |

A pharmacy's owner and superintendent may be the same person. The system must support one person holding both roles.

## 5. Scope: functional requirements

Requirement IDs are referenced in [implementation-plan.md](implementation-plan.md) and [progress.md](progress.md).

### 5.1 Customer journey: Rexa on WhatsApp (enrolment)

| ID | Requirement |
|---|---|
| CJ-1 **Scan QR code** | Each pharmacy has a printable QR poster. Scanning opens WhatsApp to the Rexa number with a pre-filled message containing the pharmacy's join code (e.g. `Hi Rexa! JOIN ADA-4821`). The join code identifies the pharmacy. |
| CJ-2 **Consent** | Rexa introduces itself as the pharmacy's assistant, explains what it will do (remind when medicine is due, answer quick questions) and what data it stores. It asks the customer to confirm their name and tap **"Confirm my details"**. No data beyond phone number and name is stored before consent. The consent version, timestamp and message ID are recorded. |
| CJ-3 **Capture medication** | Rexa asks what medicine they bought. The customer can **send a photo** of the pack or **type it**. Photos are processed by AI extraction (name, strength, form, quantity). Rexa always reads back what it understood. |
| CJ-4 **Confirm duration** | Rexa asks how long the supply lasts, using quick replies (`2 weeks`, `1 month`, `2 months`, `Other`). Free text like "6 weeks" is parsed. |
| CJ-5 **Enrolment complete** | Rexa confirms: medicine, quantity, duration, and "I'll remind you 3 days before your supply runs out". The customer is added to the retention programme and randomly assigned to an experiment arm (see 5.6). The customer can add another medicine. |
| CJ-6 **Reminder** | Rexa sends a reminder 3 days before the due date (on-time arm) or 7 days later (control arm): "Your Amlodipine 5mg is due in 3 days. Need more? Reply Yes to request a refill or No if you're all set." "Yes" creates a refill request visible on the Counter Panel. "No" closes the cycle without further reminders. |
| CJ-7 **Fallbacks** | Low-confidence photo extraction makes Rexa ask the customer to type the name. Unrecognised input twice in a row offers "Talk to the pharmacy", which flags the conversation in the Owner Workspace. |

### 5.2 Other customer interactions (any time after enrolment)

| ID | Requirement | Example trigger |
|---|---|---|
| CI-1 **Request unavailable medicine** | Customer asks if the pharmacy has a medicine. Rexa answers from a fresh stock check (≤ 2h old) or creates a stock query on the Counter Panel and replies when the attendant answers (target 2–10 min). If it is not in stock, the request goes on the "Asked for" list. | "Do you have Ozempic?" |
| CI-2 **Check stock status** | Customer asks about a medicine they previously asked for. Rexa reports the status (waiting / in transit / arrived). | "Is this in stock?" |
| CI-3 **View medication history** | Rexa lists the customer's enrolled medicines, next due dates, and recent collections. | "What do I have on record?" |
| CI-4 **Update details / change consent** | Customer can change their name, update a medicine (dose, quantity, duration), stop a medicine, pause reminders, or withdraw consent entirely (`STOP`). Withdrawal stops all messaging and triggers data deletion per [security-and-privacy.md](security-and-privacy.md). | "Change what I agree to", "STOP" |

### 5.3 Pharmacy Counter Panel

A simple tab for the attendant. **Only three actions. No login.** The device is paired once by the owner.

| ID | Requirement |
|---|---|
| CP-1 **View due list** | Two tabs: **Due (2 days)**, meaning a due date within the next 2 days, and **Overdue (≤ 7 days)**. Each row shows name, medicine + strength, quantity, and a badge if the customer replied "Yes" (refill requested). The list can be searched by name or phone so the attendant can find any enrolled customer. |
| CP-2 **Record action** | On a customer: **Collected refill** (green) or **Didn't have it** (red). "Collected" closes the current refill cycle and starts the next one. "Didn't have it" logs an unmet request on the "Asked for" list. One tap plus a confirm; undo is possible for 30 seconds. |
| CP-3 **Stock check** | Search a medicine by name or scan its barcode, then set **In stock / Low stock / Out of stock**. Shows recent checks with how long ago they were made. Pending WhatsApp stock queries ("Do you have this?") appear at the top with a timer. Answering one sends Rexa's reply to the customer. |
| CP-4 **Device pairing** | The owner generates a 6-digit pairing code in the Owner Workspace and enters it on the tablet once. The device gets a long-lived token scoped to that pharmacy. The owner can revoke it at any time. |
| CP-5 **Resilience** | Works on low-end Android tablets on 3G. Actions taken while offline are queued and synced, and the attendant can see the queued state. |

### 5.4 When a medicine is unavailable (cross-touchpoint flow)

1. A customer asks "Do you have X?", on WhatsApp (CI-1) or in person.
2. The attendant taps **"Didn't have it"** (CP-2) or marks **Out of stock** on a WhatsApp query (CP-3).
3. The item appears in the **"Asked for"** list in the Owner Workspace (OW-3), grouped by medicine with a count of customers waiting.
4. The owner optionally marks it **In transit** when ordered.
5. **When stock arrives**, the owner taps **"Now in stock"** (OW-4) on the same day (Day 0).
6. Rexa notifies every customer who asked: "Good news, Losartan 50mg is now in stock at Ada Pharmacy." (Customers who asked in person and have not enrolled are recorded by phone only if they consented at the counter; otherwise the request is counted but not notified.)

### 5.5 Owner Workspace

| ID | Requirement |
|---|---|
| OW-1 **Dashboard** | Counts for **Due soon**, **Overdue**, **Lapsed**. A **Returns (7 days after reminder)** card compares the on-time group's return rate against the 7-day-late group's. Also shows this month's incremental returns and incremental revenue in ₦. |
| OW-2 **Customers: Due / Overdue / Lapsed** | Three tabs: Due (2 days), Overdue (≤ 7 days), Lapsed (> 7 days overdue with no collection). Columns: name, medicine, due date, status. A row opens the customer's detail page (medicines, cycles, messages summary). Lists can be exported to CSV. |
| OW-3 **Asked for / Unavailable** | Unmet requests with name, medicine, date, and status (`Requested`, `In transit`, `Arrived`, `Notified`, `Fulfilled`, `Expired`). Grouped view by medicine shows the demand count. |
| OW-4 **Restock notifications** | Medicines that have arrived, with a **Notify** button that sends the in-stock message to all waiting customers, and the time of notification. |
| OW-5 **Clinical page (superintendent)** | View patient history, flag clinical alerts on a customer or medication, manage consent settings, and **veto or flag messages**: pause reminders for a customer or medicine, block a scheduled message, or request review. |
| OW-6 **Reports & invoices** | Monthly measurement report (see 5.6) and invoice, downloadable as PDF, with a payment link. |
| OW-7 **Settings** | Pharmacy profile, staff (owner, superintendent), paired devices, QR poster download, medicine catalogue (optional prices), default refill value (₦). |
| OW-8 **Flagged conversations** | Customers who asked to "Talk to the pharmacy" or hit fallbacks, with a link to open WhatsApp to them directly. |

### 5.6 Measurement & billing

Full spec: [measurement-and-billing.md](measurement-and-billing.md).

- MB-1. At first enrolment each customer is randomly assigned to the **on-time arm** (reminder at due − 3 days) or the **control arm** (reminder at due + 4 days, which is 7 days later). The default split is 80/20. Everyone is still reminded.
- MB-2. A **return** is a "Collected refill" recorded for that medication within the measurement window (the 7 days after the on-time reminder time, for both arms).
- MB-3. Incremental returns = (on-time return rate − control return rate) × number of on-time cycles. Incremental revenue (₦) = incremental returns × refill value.
- MB-4. A monthly report and invoice are generated per pharmacy. Billing is computed from the report under the pharmacy's pricing plan.

### 5.7 Support systems (internal)

| ID | Requirement |
|---|---|
| AD-1 **Admin console** | List and search pharmacies, see health (messages sent, failures, enrolments), view a pharmacy's data read-only for support, resend failed messages, manage WhatsApp templates, audit log. |
| AD-2 **Pharmacy onboarding** | Verify the pharmacy (PCN premises licence number, superintendent pharmacist's registration number, address). Create owner and superintendent accounts, generate the join code and QR poster, run setup (catalogue import, device pairing), and record that training was completed. A pharmacy cannot enrol customers until it is `verified`. |

## 6. Non-functional requirements

| Area | Requirement |
|---|---|
| Performance | Rexa replies within 3s p95 for text and within 10s p95 for photo extraction. Counter Panel actions complete within 1s p95 on 3G. |
| Reliability | Scheduled reminders: ≥ 99.5% sent within 1 hour of the scheduled time. No reminder is sent twice (idempotent sends). |
| Messaging hours | Proactive messages only between 09:00 and 19:00 WAT (Africa/Lagos). The default reminder time is 10:00 WAT. |
| Privacy | Compliant with the Nigeria Data Protection Act 2023 (NDPA). Health data is treated as sensitive personal data. See [security-and-privacy.md](security-and-privacy.md). |
| Accessibility | Counter Panel uses large tap targets (≥ 48px), high contrast, and is usable one-handed. |
| Localisation | Currency ₦ (stored in kobo). Dates shown as `26 Sep`. Phone numbers stored in E.164 (`+234…`). |
| Auditability | Every state change (consent, enrolment, reminder, action, notification, veto) is written to an append-only event log. |

## 7. Success metrics

| Metric | Target (pilot, first 90 days) |
|---|---|
| Enrolment completion rate (QR scan → enrolment complete) | ≥ 60% |
| Incremental return-rate lift (on-time − control) | ≥ 15 percentage points |
| Counter Panel recording rate (collections recorded / collections that happened, from spot audits) | ≥ 85% |
| WhatsApp stock query answered within 10 min | ≥ 80% |
| Owner workspace weekly active | ≥ 70% of paying pharmacies |
| Opt-out rate per reminder | ≤ 2% |

## 8. Assumptions and dependencies

- One shared Rexa WhatsApp Business number for all pharmacies in MVP; messages always name the pharmacy. See ADR-002 in [decisions.md](decisions.md).
- Business-initiated messages (reminders, restock alerts) use Meta-approved **utility templates**.
- Pharmacies have a tablet or Android phone with a browser at the counter.
- Attendants reliably tap "Collected refill". Measurement depends on this. Mitigated by training, a badge for requested refills, and spot audits.

## 9. Risks

| Risk | Mitigation |
|---|---|
| Attendants don't record collections, so returns are under-counted | One-tap UX, due list makes the customer obvious, weekly "unrecorded?" nudges, onboarding training |
| Small pharmacies have too few cycles for a reliable lift estimate | Show confidence intervals and fall back to a network-pooled lift for billing below minimum sample (see measurement spec) |
| WhatsApp template rejection or number quality drop | Utility-only templates, low frequency, easy opt-out, monitor quality rating |
| AI misreads a medicine from a photo | Always confirm with the customer. Low confidence falls back to typed input. Superintendent can correct |
| Ethics of a delayed-reminder control group | Control still gets reminded (7 days late, not never). Disclosed in the privacy notice. Superintendent can exempt clinically high-risk patients (excluded from measurement) |

## 10. Glossary

| Term | Definition |
|---|---|
| **Enrolment / Medication plan** | One customer + one medicine at one pharmacy, with quantity and supply duration |
| **Refill cycle** | One period of a plan, from supply start to due date and its outcome |
| **Due date** | Supply start + supply duration (the day they run out) |
| **Due (2 days)** | Due date is today or within the next 2 days |
| **Overdue** | 1–7 days past the due date with no collection |
| **Lapsed** | More than 7 days past the due date with no collection |
| **Return** | A "Collected refill" recorded within the measurement window |
| **On-time arm / control arm** | Experiment arms: reminder at due − 3d vs due + 4d (7 days later) |
| **Asked for** | An unmet request for a medicine the pharmacy did not have |
| **Join code** | Pharmacy identifier embedded in the QR (e.g. `ADA-4821`) |
