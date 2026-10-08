# Security & Privacy

RetainRx processes **health data** (what medicines someone takes), which is **sensitive personal data** under the **Nigeria Data Protection Act 2023 (NDPA)**. Treat every field about a customer's medication as sensitive.

> This document records engineering requirements, not legal advice. Items marked ⚖️ need confirmation from counsel or a licensed DPCO before the pilot launches.

## 1. Roles under the NDPA

- **Pharmacy** = data controller for its customers' data. **RetainRx** = data processor, plus controller for its own billing and analytics. ⚖️ This needs a Data Processing Agreement signed during onboarding (AD-2).
- RetainRx may need to register with the NDPC as a data controller/processor of major importance. ⚖️

## 2. Consent

- Explicit, informed, recorded consent **before** storing any medication data (CJ-2).
- Consent is versioned (`consents.version`), scoped (`reminders`, `stock_alerts`, `medication_history`), and has an evidence pointer (`wa_message_id`).
- Withdrawal is as easy as granting: `STOP` in any state, or `change_consent`.
- The privacy notice (linked in the consent message) states: what is collected, why, who sees it (the pharmacy + RetainRx), retention, rights, that **reminder timing may vary for service measurement**, and a contact.
- The superintendent can view the consent status for each customer and manage consent text settings (OW-5). Changing the consent text creates a new version, and existing customers are not silently re-consented.

## 3. Data minimisation and retention

| Data | Retention |
|---|---|
| Medicine photos (R2) | 30 days, then deleted (lifecycle rule) |
| Raw WhatsApp payloads / full message bodies | 90 days |
| Plans, cycles, consents, events | While the customer is active + 2 years after the last activity, then anonymised |
| Withdrawn customer | Personal data erased within 30 days. `events` and cycles are anonymised (customer_id replaced with an irreversible hash), so aggregate measurement survives |
| Staff accounts | Until removed + 1 year |
| Invoices | 6 years (tax) |

Erasure job: `privacy.erase_customer`. It deletes the customer, consents, conversation, messages and photos; anonymises cycles and events; and writes `privacy.erased` (without PII).

## 4. Access control

| Context | Auth | Can access |
|---|---|---|
| Customer | WhatsApp phone number (Meta-verified sender) | Their own data at that pharmacy only, via Rexa |
| Counter device | Device token (Bearer); hashed at rest, revocable | Due lists (name, medicine, qty), record actions, stock checks for **its pharmacy**. No history, no phone numbers (masked `+234 80• ••• 1234`), no clinical flags |
| Owner | Phone OTP session | Everything for their pharmacy except clinical notes marked superintendent-only; billing |
| Superintendent | Phone OTP session | Clinical page: full patient history, flags, vetoes, consent settings |
| Admin | Google OAuth + allow-list + 2FA enforced by Workspace | Cross-tenant support. **Read-only** view of pharmacy data; every access writes `admin.viewed` to `events` |

- Authorisation checks live in the API (`requireRole('owner' | 'superintendent')`, `requireDevice()`), never only in the UI.
- Postgres RLS on tenant tables as defence-in-depth.
- OTP: 6 digits, 5 min TTL, max 5 attempts, rate-limited per phone and IP. Sessions: 30 days, rotated, httpOnly + Secure + SameSite=Lax.
- Device pairing code: 6 digits, 10 min TTL, single use. Device token: 256-bit random.

## 5. Transport and storage

- TLS everywhere. HSTS on the web app.
- Encryption at rest by providers (Render Postgres, R2).
- Secrets only in environment variables, never in the repo. `.env*` is gitignored. CI runs secret scanning (gitleaks).
- Webhooks: Meta `X-Hub-Signature-256` HMAC verification; Paystack `x-paystack-signature` verification. Reject before parsing the body.

## 6. Logging

- pino redaction paths: `*.phone*`, `*.body`, `*.display_name`, `*.text`, `payload`, `authorization`, `cookie`.
- Logs and Sentry events must never contain medicine names tied to a phone or name. Use IDs.
- Sentry `beforeSend` scrubs request bodies.

## 7. AI processing

- Only the photo or text the customer sent is sent to the Claude API, with no name or phone number.
- Anthropic API data is not used for training by default. Confirm the retention terms and record them here. ⚖️
- AI output is always confirmed by the customer before it is stored as a plan.

## 8. Clinical safety

- Rexa never gives medical advice (see [conversation-design.md](conversation-design.md) §1).
- The superintendent can: hold reminders for a customer or plan, veto any scheduled message, exempt a customer from the experiment (→ always on-time), and flag clinical alerts that show on the owner's and the superintendent's view.
- Reminder copy changes require superintendent-visible release notes.

## 9. Threats to test for

- Cross-tenant access through ID guessing (every endpoint has a test using another pharmacy's IDs).
- Webhook replay / forgery.
- Counter device token theft (revocation works immediately; `last_seen_at` is visible to the owner).
- Join-code abuse (random codes, rate limit on new conversations per number).
- Prompt injection via customer text or photo text into the AI (AI output is schema-constrained and only ever proposes data that the customer must confirm; it can never trigger actions on its own).
