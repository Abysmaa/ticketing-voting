# Ticketing SaaS Cloudflare-First Architecture

## Decision

Do not use Next.js. Build a single TypeScript monorepo deployed primarily as one Cloudflare Worker with static assets. The client is React built by Vite; the API is Hono running at `/api/*` in the same Worker. This removes a second application deployment, avoids browser-to-API CORS, and supports secure same-site session cookies.

This is Cloudflare-first, not Cloudflare-only. Xendit is an external regulated payment gateway; Cloudflare hosts the web application, API, data, files, queues, coordination, scheduled jobs, and edge security. Transactional mail is sent through a provider adapter, initially Resend, from a Cloudflare Queue consumer. Cloudflare Email Service is evaluated separately because its Email Sending feature is documented as Beta.

## Why not a plain SPA plus D1 only

Ticket inventory, payment settlement, refund, and QR check-in are consistency-sensitive. A normal stateless Worker plus D1 is insufficient as the sole concurrency design for last-seat sales or two scanners reading the same ticket simultaneously. The application needs a per-event serialization boundary.

Cloudflare documents D1 as a managed serverless SQL database accessible from Workers and Pages, and Durable Objects as a stateful/distributed-systems building block. Use D1 for the durable relational record and a Durable Object per event to serialize inventory reservation and check-in decisions.

References:

- https://developers.cloudflare.com/d1/
- https://developers.cloudflare.com/durable-objects/

## Deployment topology

```text
Browser / PWA
  -> Cloudflare Worker
     -> React static assets (Vite build)
     -> Hono routes: /api/*
     -> D1: relational source of record
     -> Event Durable Object: inventory reservation and check-in serialization
     -> R2: banners, KYC documents, generated QR/PDF artifacts
     -> Queues: email, QR/PDF generation, non-critical notification work
     -> Workflows: long-running cancellation/refund and recovery orchestration
     -> Analytics Engine / Workers observability: audit metrics and traces
     -> Turnstile, WAF, rate limiting: public-edge abuse controls

Worker -> Xendit
Queue consumer -> Resend transactional email
Gateway webhook -> Worker /api/webhooks/payment
```

## Repository shape

```text
apps/web/                 React + Vite client
apps/api/                 Worker entrypoint and Hono routes
packages/contracts/       Zod schemas, API DTOs, role/permission constants
packages/domain/          pure domain rules; no Worker bindings
packages/db/              D1 migrations, query repositories, seed data
packages/ui/              shared UI primitives
workers/event-coordinator/ Durable Object for a single event
workers/jobs/             Queue consumers and Workflow definitions
tests/unit/               domain and API unit tests
tests/integration/        Worker/D1/DO integration tests
docs/                     product, architecture, ADRs, plans
wrangler.jsonc            bindings, environments, observability
```

## Service responsibilities

| Component | Use | Explicit non-use |
|---|---|---|
| Worker + Hono | API, sessions, webhook ingress, authorization, SSR-free app delivery | No mutable request state in globals |
| React + Vite | Customer, organizer, admin, and scanner client routes | No trusted business decision in browser |
| D1 | Accounts, events, orders, payments, attendees, audit/outbox records | No client-direct database access |
| Event Durable Object | Atomic inventory holds, capacity confirmation/release, idempotent check-in | Not a global object and not a general reporting database |
| R2 | public banners, private KYC documents, QR/PDF output | No public bucket access for KYC or ticket artifacts |
| Queues | retriable post-commit QR/email jobs | No financial state transition as fire-and-forget work |
| Workflows | event cancellation/refund batches, payout/KYC processes needing durable retry | Not every HTTP request |
| KV | cacheable non-sensitive configuration and public browse cache keys | Not orders, sessions, inventory, or financial facts |

## Data and consistency model

### D1 is the business record

D1 contains relational entities such as `users`, `organizers`, `events`, `event_days`, `ticket_types`, `orders`, `order_items`, `attendees`, `payments`, `payment_webhook_events`, `checkins`, `sessions`, `admin_audit_log`, and `outbox`.

Every payment webhook is persisted with a unique gateway event id before effects are applied. Replayed delivery therefore becomes an idempotent no-op. Money values are stored as integer rupiah, never floating point.

### Event Durable Object is the coordination boundary

Use deterministic object name `event:<event_id>`. Its small SQLite-backed state contains current sellable capacity, reservation expiry, and per-day check-in decision keys. It serializes these operations:

- reserve ticket quantities for a short expiry window;
- confirm or release a reservation;
- accept one check-in and reject duplicates atomically.

Do not use one global Durable Object: that becomes a bottleneck. Do not rely on Durable Object memory as durable business truth. Persist decisions, then reconcile them into D1 through an idempotent command/outbox record. Cross-store operations are not treated as one distributed transaction; failures are handled as a retryable saga with stable idempotency keys.

### Payment lifecycle

```text
DRAFT -> RESERVATION_CREATED -> PAYMENT_PENDING
  -> PAID | FREE_CONFIRMED -> TICKET_ISSUED
  -> FAILED | EXPIRED | CANCELLED | REFUNDED
```

The payment gateway webhook is authoritative. The browser return URL is presentation only and may poll an order status endpoint; it can never issue a ticket. Payment signature verification, duplicate-event protection, amount/currency comparison, and reservation confirmation happen server-side.

## Auth and authorization

- Use opaque, HttpOnly, Secure, SameSite=Lax session cookies stored server-side in D1.
- Separate role grants: attendee, organizer, admin. The same identity may hold attendee and organizer membership, but admin is internal-only.
- Google OAuth callback is handled by the Worker.
- Email/password support is a security gate: validate a Worker-compatible Argon2id/password-hashing implementation in a spike before committing to custom auth. Do not replace password hashing with fast Web Crypto primitives or invent a crypto scheme.
- Require 2FA for admin before the admin dashboard is enabled.
- Guest order access uses a one-time/rotatable signed high-entropy email token; never a guessable order id.

## Security baseline

- Verify gateway webhooks using the gateway's documented signature method and a Worker secret.
- Use a unique idempotency key for payment, refund, check-in, and payout commands.
- Rate-limit login, password reset, signup, PIN verification, promo apply, checkout start, and webhook abuse paths.
- Use Turnstile for public abuse-prone forms after UX testing.
- Generate QR secrets using cryptographic randomness; QR payload references a random token, not sequential IDs or raw PII.
- Keep private R2 objects behind signed, authorization-checked Worker routes.
- Append an admin audit record in the same D1 transaction as every administrative write.
- Enable structured logs and Worker observability; do not log PII, gateway secrets, QR tokens, or raw KYC data.

## Client route model

```text
/                         public browse
/e/:slug                  public event detail
/checkout/:reservationId  checkout
/order/:id                authenticated or signed-link order view
/tickets                  attendee account
/o                        organizer dashboard
/o/events/:id             organizer event workspace
/scan/:eventSlug          PIN-gated PWA scanner
/admin                    admin shell; 2FA required
/api/*                    same-origin Hono API
```

The UI has role-specific shells but one shared design system. Keep the scanner deliberately separate from organizer desktop UI: it is a mobile, high-contrast, one-job PWA.

## Non-negotiable implementation rules

1. No ticket is issued from a frontend success callback.
2. Never decrement quota in a browser or without the Event Durable Object.
3. Every payment, refund, and payout command is idempotent and auditable.
4. Queue consumers and Workflows are at-least-once from the application's perspective; consumers must be idempotent.
5. Public pages may be cached, but live quota, checkout, account, organizer, scanner, and admin responses must not be served from stale shared cache.
6. Do not collect or store payment-card data. Redirect/embed only through the chosen gateway's compliant flow.

## Decisions recorded and remaining gate

- Xendit is selected as the sole v1 payment gateway; its adapter and webhook rules are defined in ADR 0001.
- KYC is required before public event publishing and payout; the manual weekly payout policy is defined in ADR 0002.
- Sandbox commission, plan, tax, refund, and payout defaults are defined in ADR 0003; finance/legal approval still blocks a paid production launch.
- Resend is selected behind a transactional-email adapter in ADR 0004. Cloudflare Email Service is not the sole v1 production dependency because Email Sending is currently documented as Beta.
- Run the password-auth compatibility spike and document the result in an ADR.