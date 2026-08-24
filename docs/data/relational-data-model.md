# Ticketing SaaS Relational Data Model

## Authority boundaries

D1 is the relational business record. The event Durable Object only serializes inventory holds and check-in decisions; it never replaces order/payment history. Store all money in integer IDR, all timestamps as UTC ISO-8601, and external ids as opaque strings.

## Core tables

### Identity and access

| Table | Key columns | Invariants |
|---|---|---|
| `users` | `id`, `email`, `name`, `email_verified_at`, `created_at` | normalized email is unique |
| `password_credentials` | `user_id`, `password_hash`, `updated_at` | hash only; no plaintext/reversible secret |
| `oauth_identities` | `id`, `user_id`, `provider`, `provider_subject` | unique `(provider, provider_subject)` |
| `sessions` | `id`, `user_id`, `secret_hash`, `expires_at`, `revoked_at` | opaque cookie maps to a hashed secret |
| `organizers` | `id`, `owner_user_id`, `legal_name`, `status`, `verification_status` | one owner in v1; no team roles |
| `admins` | `user_id`, `two_factor_enabled_at`, `status` | no public creation route |

### Events and sellable inventory

| Table | Key columns | Invariants |
|---|---|---|
| `events` | `id`, `organizer_id`, `slug`, `status`, `category`, `created_at` | unique slug; only verified organizer can become `published` |
| `event_days` | `id`, `event_id`, `starts_at`, `ends_at`, `venue`, `timezone` | end after start |
| `ticket_types` | `id`, `event_id`, `name`, `price_idr`, `quota`, `sale_start_at`, `sale_end_at` | price >= 0; quota > 0; sale window valid |
| `ticket_type_days` | `ticket_type_id`, `event_day_id` | defines day-specific pass; none means full-pass |
| `inventory_reservations` | `id`, `event_id`, `expires_at`, `status`, `idempotency_key` | durable projection/audit of DO hold; unique client idempotency key |
| `inventory_reservation_items` | `reservation_id`, `ticket_type_id`, `quantity` | quantity > 0 |

### Order, payment, and ticket

| Table | Key columns | Invariants |
|---|---|---|
| `orders` | `id`, `event_id`, `buyer_user_id?`, `buyer_email`, `status`, `currency`, `subtotal_idr`, `discount_idr`, `tax_idr`, `total_idr` | immutable money snapshot after payment begins |
| `order_items` | `id`, `order_id`, `ticket_type_id`, `unit_price_idr`, `quantity` | ticket type snapshot preserved even after later edits |
| `payments` | `id`, `order_id`, `provider`, `provider_payment_id`, `status`, `expected_amount_idr` | unique `(provider, provider_payment_id)` |
| `payment_webhook_events` | `id`, `provider`, `provider_event_id`, `received_at`, `processed_at` | unique `(provider, provider_event_id)` gives replay protection |
| `attendees` | `id`, `order_item_id`, `name`, `email`, `qr_token_hash`, `ticket_status` | one row per admitted person; QR token unique and non-sequential |
| `order_access_tokens` | `id`, `order_id`, `token_hash`, `expires_at`, `revoked_at` | signed guest-email access, never raw order enumeration |

### Operations and compliance

| Table | Key columns | Invariants |
|---|---|---|
| `organizer_kyc_submissions` | `id`, `organizer_id`, `status`, `document_object_key`, `reviewed_by_admin_id?` | only private R2 keys, no public URL |
| `payouts` | `id`, `organizer_id`, `amount_idr`, `status`, `reference`, `marked_paid_by_admin_id?` | payout only for verified organizer; unique external reference |
| `checkins` | `id`, `attendee_id`, `event_day_id?`, `scanned_at`, `scanner_session_id` | unique `(attendee_id, event_day_id)`; single-day uses normalized day key |
| `admin_audit_log` | `id`, `admin_id`, `action`, `target_type`, `target_id`, `reason`, `created_at` | written in same D1 transaction as admin mutation |
| `outbox` | `id`, `type`, `aggregate_id`, `payload_json`, `idempotency_key`, `processed_at` | created in the same transaction as state transition |

## Required status machines

```text
Event: draft -> published -> closed | cancelled
Order: reservation_created -> payment_pending -> paid | free_confirmed | expired | cancelled | refunded
Payment: created -> pending -> succeeded | failed | expired | refunded
KYC: unverified -> pending -> verified | rejected
Payout: pending -> approved -> paid | failed | cancelled
```

A status transition must be validated in domain code; do not accept arbitrary client-supplied status values.

## Indexes and constraints required before pilot

- `events(slug) UNIQUE`
- `events(organizer_id, status)`
- `ticket_types(event_id, sale_start_at, sale_end_at)`
- `orders(event_id, status, created_at)`
- `payments(provider, provider_payment_id) UNIQUE`
- `payment_webhook_events(provider, provider_event_id) UNIQUE`
- `checkins(attendee_id, event_day_id) UNIQUE`
- `outbox(idempotency_key) UNIQUE`
- `admin_audit_log(target_type, target_id, created_at)`

## Rules that belong outside plain SQL

- Ticket availability and holds are decided by the event Durable Object.
- Payment webhook signature and amount validation are in the Worker adapter.
- Admin 2FA/role authorization occurs before a query is allowed to reveal KYC or payout data.
- R2 object access is authorized by Worker code; a database row alone must not expose a direct object URL.
