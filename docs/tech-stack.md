# Tech Stack

Chosen for a small team shipping a pilot in Nigeria. The priorities are **one language end-to-end**, **few moving parts**, and **cheap to run**. Rationale for the non-obvious choices is in [decisions.md](decisions.md).

## Summary

| Layer | Choice | Why |
|---|---|---|
| Language | **TypeScript 5.x (strict)** on **Node.js 22 LTS** | One language across API, worker and web; shared types and Zod schemas |
| Monorepo | **pnpm workspaces + Turborepo** | Shared `packages/*` (db, domain, ui), cached builds |
| API | **Fastify 5** + **Zod** (`fastify-type-provider-zod`) | Fast, schema-first, good for webhooks |
| Web | **Next.js 15 (App Router)** + **React 19** | Owner Workspace, Clinical page, Admin console, Counter Panel (PWA) in one app |
| UI | **Tailwind CSS 4** + **shadcn/ui** (Radix) + **lucide-react** | Fast, accessible components matching the clean card UI in the design |
| Data fetching | **TanStack Query** | Caching and the offline mutation queue for the Counter Panel |
| Charts | **Recharts** | Dashboard return-rate bars and trends |
| Database | **PostgreSQL 16** | Single source of truth; relational data with strong constraints |
| ORM / migrations | **Drizzle ORM** + **drizzle-kit** | Type-safe SQL, plain SQL migrations committed to the repo |
| Jobs / scheduling | **pg-boss** (Postgres-backed queue) | Scheduled reminders, retries and cron jobs without adding Redis |
| WhatsApp | **Meta WhatsApp Business Cloud API** (direct, Graph API) | Official; templates, interactive buttons, media, webhooks |
| AI | **Anthropic Claude API** (`@anthropic-ai/sdk`): `claude-sonnet-5-5` for photo extraction, `claude-haiku-4-5` for intent classification and free-text parsing | Strong vision plus structured output via tool use; Haiku is cheap and fast for intents |
| Object storage | **Cloudflare R2** (S3-compatible) | Medicine photos (short retention) and generated PDFs |
| Auth (owner/superintendent) | Phone + **OTP via WhatsApp authentication template**, **Termii SMS** fallback; httpOnly session cookie | Nigerian owners live on WhatsApp; no passwords |
| Auth (counter device) | Device pairing code → hashed long-lived device token | "No login" requirement |
| Auth (admin) | Google Workspace OAuth + email allow-list | Internal only |
| Payments | **Paystack** (payment links / invoices, bank transfer) | Nigerian ₦ payments |
| PDF | **@react-pdf/renderer** | Reports, invoices and QR posters |
| QR | **qrcode** (npm) | Join-code QR posters |
| Barcode scan | **@zxing/browser** | Camera barcode scanning on the counter tablet |
| Dates | **date-fns** + **@date-fns/tz** | All business logic in `Africa/Lagos` |
| Validation | **Zod** | Shared request/response/env schemas |
| Logging | **pino** (JSON) | Structured logs with PII redaction |
| Errors / APM | **Sentry** | API, worker and web |
| Testing | **Vitest** (unit/integration), **Playwright** (e2e), **Testcontainers** (Postgres) | Fast tests against a real database |
| Lint / format | **ESLint** (typescript-eslint) + **Prettier** | Enforced in CI |
| CI | **GitHub Actions** | Lint, typecheck, test, migration check |
| Hosting | **Render**: web service (api), web service (web), background worker (worker), managed Postgres | Simple PaaS; EU (Frankfurt) region in MVP. Data-residency review is in [decisions.md](decisions.md) |
| Secrets | Render environment groups; `.env` locally (never committed) | |

## Repository layout

```
retainrx/
├── apps/
│   ├── api/            # Fastify: REST API, WhatsApp webhook, Paystack webhook
│   ├── worker/         # pg-boss consumers: reminders, notifications, measurement, invoicing
│   └── web/            # Next.js: (owner) (clinical) (admin) (counter) route groups
├── packages/
│   ├── db/             # Drizzle schema, migrations, seed
│   ├── domain/         # Pure business logic: cycle status, arm assignment, metrics, pricing
│   ├── rexa/           # Conversation engine: state machine, intents, copy, templates
│   ├── whatsapp/       # Cloud API client, webhook verification, template registry
│   ├── ai/             # Claude client wrappers: medicine extraction, intent classification
│   ├── ui/             # Shared React components (shadcn-based)
│   └── config/         # Zod env schema, eslint/tsconfig presets
├── docs/               # This spec
├── CLAUDE.md           # Rules for AI coding agents
└── turbo.json
```

## Environment variables

Validated at boot by `packages/config` (the app refuses to start if any are missing).

```
DATABASE_URL=
APP_BASE_URL=                     # e.g. https://app.retainrx.ng
API_BASE_URL=
SESSION_SECRET=
DEVICE_TOKEN_PEPPER=

WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_BUSINESS_ACCOUNT_ID=
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_APP_SECRET=              # webhook signature (X-Hub-Signature-256)
WHATSAPP_VERIFY_TOKEN=            # webhook subscription handshake
REXA_WA_NUMBER=                   # E.164, used to build wa.me links

ANTHROPIC_API_KEY=
AI_MODEL_VISION=claude-sonnet-5-5
AI_MODEL_FAST=claude-haiku-4-5

R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=

TERMII_API_KEY=
TERMII_SENDER_ID=

PAYSTACK_SECRET_KEY=
PAYSTACK_WEBHOOK_SECRET=

GOOGLE_OAUTH_CLIENT_ID=
GOOGLE_OAUTH_CLIENT_SECRET=
ADMIN_EMAIL_ALLOWLIST=

SENTRY_DSN=
TZ_BUSINESS=Africa/Lagos
```

## Version policy

- Pin exact versions in `package.json`; Renovate opens weekly grouped PRs.
- Node version pinned in `.nvmrc` and `engines`.
- Adding a new runtime dependency needs one line of justification in the PR description. Prefer what is already in this list.
