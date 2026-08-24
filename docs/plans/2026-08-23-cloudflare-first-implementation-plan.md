# Ticketing SaaS Cloudflare-First Implementation Plan

> **For Hermes:** Execute this plan only after the business decisions listed in Milestone 0 are resolved. Preserve the existing product specs and wireframes as inputs; do not silently convert their open assumptions into production policy.

**Goal:** Deliver a production-oriented ticketing MVP on Cloudflare without Next.js, beginning with a validated foundation and ending with one paid or free ticket flow that can be checked in safely.

**Architecture:** React/Vite client and Hono API in one Cloudflare Worker; D1 holds relational facts, an event-scoped Durable Object serializes capacity/check-in, R2 stores artifacts, and Queues/Workflows own retryable async work.

**Tech stack:** TypeScript, React, Vite, Hono, Zod, Cloudflare Workers, D1, R2, Queues, Durable Objects, Workflows, Vitest with the Cloudflare Workers pool.

---

## Milestone 0: Lock product and technical gates

**Objective:** Remove decisions that would otherwise cause financial or security rework.

**Documents:**
- Update: `PRODUCT_SPEC.md`
- Update: `TRD.md`
- Create: `docs/adr/0001-v1-payment-gateway.md`
- Create: `docs/adr/0002-auth-and-password-hashing.md`
- Create: `docs/adr/0003-money-tax-payout-policy.md`

**Tasks:**
1. Implement the selected Xendit adapter and webhook boundary from `docs/adr/0001-xendit-payment-gateway.md`.
2. Implement the KYC-before-publication and manual weekly payout policy from `docs/adr/0002-organizer-kyc-and-payout-policy.md`.
3. Use sandbox commercial settings only from `docs/adr/0003-sandbox-commercial-policy.md`; keep paid production deployment blocked pending finance/legal approval.
4. Run a Worker-compatible password-hashing spike; reject any option that does not provide a secure maintained password-hash implementation.
5. Implement Resend through the adapter and Queue delivery contract in `docs/adr/0004-transactional-email.md`.
6. Reconcile the admin scope in `ROADMAP.md` against `ADMIN_SPEC.md`.

**Verification:** Every decision has owner, date, and reason. No payment or custom password code begins before approval.

## Milestone 1: Establish the Cloudflare application foundation

**Objective:** Create a typed monorepo that deploys a static React application and same-origin API to a Worker.

**Files:**
- Create: `package.json`, `pnpm-workspace.yaml`, `tsconfig.json`, `wrangler.jsonc`
- Create: `apps/web/`, `apps/api/`, `packages/contracts/`, `packages/domain/`, `packages/db/`, `tests/`
- Create: `.github/workflows/ci.yml`
- Create: `.dev.vars.example`

**Tasks:**
1. Scaffold Vite React client in `apps/web`.
2. Add Hono Worker entrypoint in `apps/api` with `/api/health`.
3. Configure Worker static assets from the Vite build, environment bindings, `nodejs_compat`, current compatibility date, and structured observability.
4. Generate Worker binding types from Wrangler configuration; do not hand-write `Env`.
5. Add Vitest Worker-pool configuration and a health-route test.
6. Add CI steps for install, typecheck, lint, test, and production build.

**Verification:** Local Worker serves `/`, `/api/health`, and a passing Worker-pool test. Preview deployment succeeds with no production secrets.

## Milestone 2: Model D1 facts and role-safe sessions

**Objective:** Create migration-backed relational data and secure session/role boundaries.

**Files:**
- Create: `packages/db/migrations/0001_identity.sql`
- Create: `packages/db/migrations/0002_events_orders.sql`
- Create: `apps/api/src/auth/*`
- Create: `apps/api/src/middleware/require-role.ts`
- Test: `tests/integration/auth.test.ts`

**Tasks:**
1. Add migrations for users, organizer membership, admin membership, sessions, events, event days, ticket types, orders, order items, attendees, payment events, audit logs, and outbox records.
2. Store rupiah values as integer columns and define unique idempotency constraints.
3. Build opaque session create/read/revoke helpers with secure cookie attributes.
4. Implement organizer/attendee role guards and internal-only admin provisioning.
5. Implement Google OAuth only after callback and session tests exist.
6. Add password login only after the Milestone 0 security spike is accepted.

**Verification:** Unauthorized users cannot access organizer/admin routes; revoked sessions fail; migrations apply to local and remote preview D1.

## Milestone 3: Make the wireframes implementable before styling

**Objective:** Convert coverage-map wireframes into named, testable UI states and a real route map.

**Files:**
- Create: `packages/contracts/src/view-models/*`
- Create: `apps/web/src/routes/*`
- Create: `apps/web/src/features/public-event/*`
- Update: `docs/product/2026-08-23-wireframe-audit.md`

**Tasks:**
1. Split each contradictory artboard into one primary scenario and separate variants.
2. Start with the buyer slice: browse, event detail, sold-out, cart, payment pending/failed, confirmation.
3. Add explicit loading, empty, validation-error, network-error, cancelled, and unauthorized states.
4. Replace visual placeholders with typed fixtures that match API DTOs.
5. Add keyboard, mobile, and screen-reader acceptance criteria for checkout and scanner states.

**Verification:** Story/test coverage demonstrates that no screen renders mutually exclusive state combinations.

## Milestone 4: Build inventory reservation and free-registration vertical slice

**Objective:** Let an attendee reserve an in-stock free ticket, confirm it, receive a QR artifact, and check in once.

**Files:**
- Create: `workers/event-coordinator/src/event-coordinator.ts`
- Create: `apps/api/src/routes/events.ts`
- Create: `apps/api/src/routes/orders.ts`
- Create: `apps/api/src/routes/checkin.ts`
- Create: `workers/jobs/src/issue-ticket-consumer.ts`
- Test: `tests/integration/reservation-and-checkin.test.ts`

**Tasks:**
1. Add deterministic Event Durable Object binding and migration.
2. Test an atomic capacity hold, expiry, confirmation, and duplicate command retry.
3. Create draft/publish event and ticket-type APIs with organizer authorization.
4. Implement `POST /orders` free-confirmed path through a short-lived reservation.
5. Create attendee QR token and enqueue ticket issuance only after confirmation.
6. Implement PIN-gated scanner validation and atomic check-in duplicate rejection.

**Verification:** Parallel attempts for the final ticket produce exactly one confirmation; the same QR can check in once per valid event day.

## Milestone 5: Add selected gateway with webhook-authoritative payment

**Objective:** Turn the free vertical slice into a paid flow without trusting browser callbacks.

**Files:**
- Create: `apps/api/src/payments/gateway.ts`
- Create: `apps/api/src/payments/<selected-gateway>.ts`
- Create: `apps/api/src/routes/payment-webhooks.ts`
- Test: `tests/integration/payment-webhook.test.ts`

**Tasks:**
1. Implement a gateway adapter behind a narrow payment-intent interface.
2. Add payment attempt and webhook event persistence before any status mutation.
3. Verify gateway signatures using Worker secrets.
4. Confirm reservation and transition order to paid only from valid webhook handling.
5. Add expiry/release flow and safe pending/failed UI polling.
6. Test duplicate, out-of-order, invalid-signature, wrong-amount, and late webhook cases.

**Verification:** A valid webhook issues one ticket once; replayed or invalid webhooks never issue a duplicate ticket or alter money facts.

## Milestone 6: Organizer operations and guarded administration

**Objective:** Give organizers event visibility and give internal operators auditable KYC/payout controls.

**Files:**
- Create: `apps/api/src/routes/organizer/*`
- Create: `apps/api/src/routes/admin/*`
- Create: `apps/web/src/features/organizer/*`
- Create: `apps/web/src/features/admin/*`
- Test: `tests/integration/admin-audit.test.ts`

**Tasks:**
1. Implement sales summary, attendee search/export, single-order refund, and whole-event cancellation workflow.
2. Implement private KYC upload through authorization-checked R2 routes.
3. Require KYC verification before payout action.
4. Make admin 2FA, reason capture, confirmation, and audit transaction mandatory for KYC, suspension, refund, and payout writes.
5. Use a Workflow for batch event cancellation/refunds with resume-safe idempotency.

**Verification:** Every admin mutation has a matching audit record; sensitive R2 objects cannot be fetched anonymously; batch retries do not double-refund.

## Milestone 7: Hardening and pilot launch

**Objective:** Prove the system is safe enough for one controlled event.

**Tasks:**
1. Add rate limits, Turnstile where justified, CSP/security headers, and PII-safe observability.
2. Run capacity-race, scanner-duplicate, webhook-replay, and payment-expiry tests.
3. Test mobile scanner camera denied/offline/manual-entry scenarios.
4. Publish legal route placeholders only after business-provided policy text is approved.
5. Run a single organizer/single event pilot with operational runbook, alert ownership, and rollback procedure.

**Verification:** The pilot checklist passes and all unresolved business decisions are explicitly accepted or block launch.

## Commit policy

Commit each completed milestone or independently testable slice. Do not commit generated secrets, real KYC documents, production payment payloads, or production QR tokens.