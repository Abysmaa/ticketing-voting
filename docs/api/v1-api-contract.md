# Ticketing SaaS API Contract v1

## Conventions

- Base path: `/api/v1`.
- JSON request/response. All money is integer `*_idr`.
- Browser authentication: secure HttpOnly session cookie. No access token stored in localStorage.
- Mutating payment, reservation, refund, KYC, and payout routes require an `Idempotency-Key` header.
- Error shape:

```json
{ "error": { "code": "TICKET_SOLD_OUT", "message": "This ticket is no longer available.", "request_id": "..." } }
```

## Public event and checkout

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/events` | none | Browse only published/on-sale events |
| `GET` | `/events/:slug` | none | Public detail; live availability is advisory only |
| `POST` | `/events/:eventId/reservations` | optional | Create short ticket hold through Event Durable Object |
| `GET` | `/reservations/:id` | owner/guest secret | Show expiry and cart details |
| `POST` | `/orders` | optional | Convert valid reservation into free confirmation or Xendit payment request |
| `GET` | `/orders/:id` | order owner or signed access token | Read order/ticket/invoice state |
| `POST` | `/orders/:id/access-token` | none | Request a new rate-limited signed email link |

`POST /orders` must reject expired reservation, changed price, sold-out capacity, invalid promo, and tampered totals. The API computes totals; the client never sends a trusted final amount.

## Payment webhook

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `POST` | `/webhooks/xendit` | Xendit signature only | Persist and process authoritative payment event |

Webhook processing order: verify signature -> dedupe provider event id -> validate expected order amount/currency -> transactional order/payment transition -> outbox ticket issuance -> acknowledge. Browser callback is not a substitute for this endpoint.

## Authentication

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/auth/signup` | create attendee or organizer identity |
| `POST` | `/auth/login` | create session |
| `POST` | `/auth/logout` | revoke current session |
| `GET` | `/auth/google/start` | begin Google OAuth |
| `GET` | `/auth/google/callback` | finish OAuth and create session |
| `POST` | `/auth/password-reset/request` | rate-limited reset email |
| `POST` | `/auth/password-reset/confirm` | consume one-time token |
| `GET` | `/me` | current identity and role grants |

Admin is excluded from public signup. Admin routes require a verified second factor.

## Organizer

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/organizer/events` | create draft |
| `PATCH` | `/organizer/events/:id` | edit owned draft/event under business rules |
| `POST` | `/organizer/events/:id/publish` | requires verified KYC |
| `POST` | `/organizer/events/:id/cancel` | starts idempotent refund Workflow |
| `POST` | `/organizer/events/:id/ticket-types` | create ticket type |
| `GET` | `/organizer/events/:id/attendees` | paginated list or streamed CSV export |
| `POST` | `/organizer/orders/:id/refund` | full/supported refund action |
| `POST` | `/organizer/kyc-submissions` | submit protected KYC object reference |
| `GET` | `/organizer/dashboard` | sales and event summary |

## Scanner

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/scanner/sessions` | verify event PIN and create short scanner session |
| `POST` | `/scanner/sessions/:id/validate` | validate QR for selected event day |
| `POST` | `/scanner/sessions/:id/checkins` | atomically accept/reject check-in |

Responses must distinguish `ACCEPTED`, `ALREADY_CHECKED_IN`, `WRONG_EVENT_DAY`, `INVALID_TICKET`, `CANCELLED_TICKET`, and `OFFLINE_UNAVAILABLE`.

## Admin

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/admin/organizers` | search/filter organizer operations view |
| `POST` | `/admin/kyc/:id/approve` | approve KYC with audit reason |
| `POST` | `/admin/kyc/:id/reject` | reject KYC with required reason |
| `POST` | `/admin/organizers/:id/suspend` | suspend with required reason |
| `POST` | `/admin/payouts/:id/mark-paid` | record external transfer reference |
| `GET` | `/admin/audit-log` | filterable immutable activity history |

Every successful admin mutation writes an `admin_audit_log` entry in the same D1 transaction.
