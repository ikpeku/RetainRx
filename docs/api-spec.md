# API Specification

Base URL: `{API_BASE_URL}/v1`. JSON over HTTPS. Request and response bodies are defined as Zod schemas in `apps/api/src/routes/**/schema.ts`, and an OpenAPI document is generated from them at `/v1/openapi.json`.

## Conventions

- Auth headers: staff session cookie `rrx_session`; counter `Authorization: Bearer <device_token>`; admin cookie `rrx_admin`.
- Tenant is derived from auth and **never** accepted in the path or body.
- Errors: `{ "error": { "code": "STRING_CODE", "message": "human text", "details"?: {} } }` with the right HTTP status.
- Pagination: cursor-based, `?cursor=&limit=` (default 50, max 200) → `{ items, nextCursor }`.
- Idempotency: mutating counter endpoints require `client_action_id` (uuid).
- Dates: `YYYY-MM-DD` (Lagos business date). Instants: ISO-8601 UTC.
- Money: integer kobo in fields ending `_kobo`.

## Webhooks (no auth; signature verified)

| Method | Path | Notes |
|---|---|---|
| GET | `/webhooks/whatsapp` | Meta subscription verify (`hub.verify_token`) |
| POST | `/webhooks/whatsapp` | Messages + statuses. Verify signature → persist → enqueue → 200 |
| POST | `/webhooks/paystack` | `charge.success` → mark invoice paid |

## Staff auth

| Method | Path | Body / notes |
|---|---|---|
| POST | `/auth/otp/request` | `{ phone }` → sends WhatsApp OTP (SMS fallback). Always 202 (no user enumeration) |
| POST | `/auth/otp/verify` | `{ phone, code }` → sets session; returns `{ staff, pharmacies:[{id,name,roles}] }` |
| POST | `/auth/logout` | |
| POST | `/auth/switch-pharmacy` | `{ pharmacy_id }` (must hold a role there) |

## Counter Panel (device token)

| Method | Path | Notes |
|---|---|---|
| POST | `/counter/pair` | `{ code, label }` (no auth) → `{ device_token }` shown once |
| GET | `/counter/due?tab=due\|overdue&q=` | Rows: `{ plan_id, cycle_id, customer_name, phone_masked, medicine, strength, quantity, unit, due_date, status, refill_requested }` |
| GET | `/counter/customers?q=` | Search enrolled customers (name / last 4 digits of phone) |
| POST | `/counter/actions` | `{ client_action_id, plan_id, action: 'collected'\|'didnt_have' }` → `{ action_id, undo_until }` |
| POST | `/counter/actions/:id/undo` | Within 30s |
| GET | `/counter/medicines?q=&barcode=` | Search catalogue |
| POST | `/counter/medicines` | Quick-add `{ name, strength?, form?, barcode? }` |
| GET | `/counter/stock-checks/recent` | Last 20 with `checked_at` |
| POST | `/counter/stock-checks` | `{ client_action_id, medicine_id, status, stock_query_id? }` |
| GET | `/counter/stock-queries?status=pending` | Pending WhatsApp "Do you have this?" with `age_seconds` |
| POST | `/counter/walk-in-requests` | `{ client_action_id, medicine_id, customer_phone? }`, for a walk-in asking for something unavailable |

## Owner Workspace (session, role `owner`)

| Method | Path | Notes |
|---|---|---|
| GET | `/workspace/dashboard` | `{ due_soon, overdue, lapsed, returns: { ontime_rate, control_rate, lift, window:'rolling_30d' }, month: { incremental_returns, incremental_revenue_kobo, lift_source } }` |
| GET | `/workspace/customers?tab=due\|overdue\|lapsed\|all&q=&cursor=` | |
| GET | `/workspace/customers/:id` | Profile, plans, cycles, consent status, message summary (no raw bodies) |
| GET | `/workspace/customers/export?tab=` | CSV |
| GET | `/workspace/asked-for?status=&group_by=medicine` | |
| PATCH | `/workspace/asked-for/:id` | `{ status: 'in_transit'\|'cancelled' }` |
| POST | `/workspace/medicines/:id/now-in-stock` | Marks waiting requests `arrived`, creates `restock_notification`. `{ notify: boolean }` |
| POST | `/workspace/restock-notifications/:id/notify` | Sends `back_in_stock_v1` to waiting customers |
| GET | `/workspace/restock-notifications` | |
| GET | `/workspace/flagged-conversations` | |
| POST | `/workspace/flagged-conversations/:customer_id/resolve` | |
| GET | `/workspace/reports?period=YYYY-MM` | Snapshot + report data |
| GET | `/workspace/invoices` · `/workspace/invoices/:id/pdf` | |
| GET/PATCH | `/workspace/settings` | Profile, reminder time, default refill value |
| GET/POST/DELETE | `/workspace/staff` | Invite superintendent / co-owner by phone |
| POST | `/workspace/devices/pairing-code` | → `{ code, expires_at }` |
| GET · DELETE | `/workspace/devices` · `/workspace/devices/:id` | List / revoke |
| GET | `/workspace/qr-poster.pdf` | |
| GET/POST/PATCH | `/workspace/medicines` | Catalogue + prices |

## Clinical page (session, role `superintendent`)

| Method | Path | Notes |
|---|---|---|
| GET | `/clinical/patients?q=` | |
| GET | `/clinical/patients/:id/history` | Plans, cycles, collections, messages (bodies included, access logged) |
| POST | `/clinical/flags` | `{ customer_id, plan_id?, kind, note }` |
| PATCH | `/clinical/flags/:id` | resolve |
| GET | `/clinical/scheduled-messages?from=&to=` | Upcoming messages for review |
| POST | `/clinical/scheduled-messages/:id/veto` | `{ reason }` |
| GET/PATCH | `/clinical/consent-settings` | Scopes offered, privacy notice URL, consent version |

## Admin (admin session)

| Method | Path | Notes |
|---|---|---|
| GET/POST | `/admin/pharmacies` | Create during onboarding |
| GET/PATCH | `/admin/pharmacies/:id` | Status, verification fields, pricing plan |
| POST | `/admin/pharmacies/:id/verify` | Requires licence + superintendent PCN no. |
| POST | `/admin/pharmacies/:id/staff` | Create owner/superintendent |
| GET | `/admin/pharmacies/:id/health` | Sends, failures, enrolments, stock query SLA |
| GET | `/admin/messages?status=failed` | |
| POST | `/admin/messages/:id/retry` | |
| GET/POST | `/admin/invoices` · `/admin/invoices/:id/issue` · `/void` | |
| GET | `/admin/events?pharmacy_id=&type=` | Audit log |
| GET/PATCH | `/admin/templates` | Template registry status |

## Internal jobs (pg-boss queue names)

| Queue | Trigger |
|---|---|
| `rexa.handle_inbound` | per inbound message |
| `whatsapp.send` | per outbound message |
| `reminders.dispatch` | cron `*/5 * * * *` |
| `stock_queries.timeout` | cron `* * * * *` |
| `stock_requests.expire` | cron daily 03:00 WAT |
| `measurement.recompute` | cron daily 02:00 WAT |
| `billing.generate_invoices` | cron 06:00 WAT on day 9 |
| `privacy.erase_customer` | on withdrawal (+ grace) |
| `retention.purge` | cron daily 04:00 WAT (photos, payloads) |
