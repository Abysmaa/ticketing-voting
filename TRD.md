# Ticketing SaaS — Technical Requirements Document (TRD) v1

Derived from `PRODUCT_SPEC.md` (PRD). This doc turn product scope into build-able tech spec.

## 1. Goal
Answer *how* to build what PRD defines as MVP. Target: single-tenant ticketing platform, web checkout + web-based check-in scanner, Indonesia payment gateway.

## 2. Architecture Overview
- **Web app** (organizer dashboard + attendee checkout) — React SPA built with Vite and served as static assets from the same Cloudflare Worker.
- **API/backend** — Hono REST API in that Worker, serving the web app and browser-based PWA scanner.
- **DB** — Cloudflare D1 is the relational business record. It stores accounts, events, orders, payments, attendees, audit log, and idempotency/outbox records.
- **Coordination** — one Durable Object per event serializes inventory reservation and duplicate check-in decisions; it is not the sole business record.
- **Async** — Cloudflare Queues for QR/email delivery and post-commit work; Cloudflare Workflows for resilient batch cancellation/refund operations.
- **Storage** — Cloudflare R2 for event assets, KYC documents, and generated QR/PDF artifacts.
- **Payment gateway integration** — Xendit behind an API adapter; buyer only ever sees payment METHODS (VA/e-wallet/QRIS/card), never a gateway name. Webhook callback drives payment status. See `docs/adr/0001-xendit-payment-gateway.md`.
- **Check-in scanner** — PWA, browser camera access (`getUserMedia` / barcode-detector API), with server-authoritative validation and idempotent check-in.

```text
[Attendee/Organizer Browser] -> [Worker: static React + Hono API] -> [D1 business record]
                                         |         |       \
                                         |         |        -> [Queues] -> [Resend email/QR]
                                         |         -> [Event Durable Object]
                                         -> [R2 private/public assets]
[Xendit Webhook] -----------------------> [Worker: verified, idempotent payment transition]
[Check-in Scanner PWA] -----------------> [Event Durable Object + D1 check-in record]
```

## 3. Tech Stack (accepted v1 direction)
- Runtime/API: Cloudflare Workers + Hono, one deployment with same-origin API.
- Frontend: React + Vite. **No Next.js.**
- DB: Cloudflare D1.
- Coordination: Cloudflare Durable Objects, one deterministic object per event.
- Async: Cloudflare Queues and Workflows.
- File/asset storage: Cloudflare R2.
- Payment: Xendit adapter (ADR 0001).
- Auth: opaque server-side session cookie in D1. Both Organizer and Attendee get full account: **email + password** signup/login, plus **Google OAuth** social login. Separate role grants, same auth mechanism/UI pattern. Attendee checkout still offers **guest-checkout** (no login required to buy) alongside "Log In / Sign Up" — logging in at checkout ties the order to the account so it shows in "My Tickets"; a pure guest order is reachable only via the emailed confirmation link, not the dashboard.
- Forgot/reset password: standard email-link token flow (request -> emailed link -> set new password).
- Email: Resend behind a transactional-email adapter, invoked from idempotent Queue consumers after D1 commit (ADR 0004).

## 4. Data Model (expand PRD sect. 6 into schema draft)

### `organizer`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| name | text | |
| email | text unique | |
| plan_tier | enum(free, paid) | matches PRD decision |
| commission_rate | numeric | % override, default from platform config |
| verification_status | enum(unverified, pending, verified) | KYC status — payout gateway requires this before disbursement |
| tax_id | text nullable | NPWP or business registration number, collected during verification |
| created_at | timestamp | |

### `event`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| organizer_id | uuid FK -> organizer | |
| title | text | |
| description | text | |
| banner_url | text | |
| type | enum(single_day, multi_day) | **structural** shape (day count/pass logic) — separate from `category` below |
| category | enum(conference, workshop, festival, meetup, other) | browse/filter taxonomy (Homepage category chip) — independent of `type` |
| status | enum(draft, published, cancelled, closed) | `cancelled` = organizer-initiated, triggers mass refund of all order for this event |
| created_at | timestamp | |

### `event_day`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| event_id | uuid FK -> event | |
| date | date | |
| venue | text | |
| timezone | text | IANA name, e.g. `Asia/Jakarta` (WIB) / `Asia/Makassar` (WITA) / `Asia/Jayapura` (WIT) — per-day since multi-city multi-day event possible |

### `ticket_type`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| event_id | uuid FK -> event | |
| name | text | |
| price | numeric | 0 = free/registration |
| quota | int | |
| sold_count | int | derived or cached |
| valid_days | uuid[] | FK -> event_day, null = full-pass |
| sale_start | timestamp | |
| sale_end | timestamp | |

### `order`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| event_id | uuid FK -> event | |
| buyer_name | text | |
| buyer_email | text | |
| status | enum(pending, paid, free_confirmed, cancelled, refunded) | |
| subtotal_amount | numeric | pre-tax ticket total |
| tax_amount | numeric | Sandbox default is 0. Production tax/remitter policy is a finance/legal launch gate; do not label an invoice PPN-inclusive until it is approved. |
| total_amount | numeric | subtotal + tax − discount |
| commission_amount | numeric | computed at order time from organizer commission_rate, on **subtotal** (pre-tax) — confirm w/ finance |
| payment_gateway | enum(xendit, none) | none = free ticket |
| payment_ref | text | gateway transaction id |
| created_at | timestamp | |

### `attendee`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| order_id | uuid FK -> order | |
| ticket_type_id | uuid FK -> ticket_type | |
| name | text | |
| email | text | |
| qr_code | text unique | |
| checked_in_at | timestamp nullable | |

### `promo_code`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| event_id | uuid FK -> event, nullable | null = platform-wide (unlikely v1) |
| ticket_type_id | uuid FK, nullable | scope to specific ticket type |
| code | text unique | |
| discount_type | enum(percentage, flat) | |
| discount_value | numeric | |
| max_uses | int | |
| used_count | int | |

### `check_in`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| attendee_id | uuid FK -> attendee | |
| event_day_id | uuid FK -> event_day, nullable | which day this check-in is for (multi-day festival); null for single-day event |
| scanned_by | text | device/session identifier (see Check-in Access below — no staff account in v1) |
| scanned_at | timestamp | |
| gate | text nullable | |

### `admin`
*Full admin page list, requirements, and additional entities (audit log, payout record, dispute log): see `ADMIN_SPEC.md`.*
| field | type | note |
|---|---|---|
| id | uuid PK | |
| name | text | |
| email | text unique | |
| created_at | timestamp | |

**No public signup** — rows created directly by the platform team (seed script / internal tool), never via `POST /auth/signup`. Separate auth pool from organizer/attendee; same session mechanism, `role: admin` on the session token.

### `event_checkin_access`
| field | type | note |
|---|---|---|
| id | uuid PK | |
| event_id | uuid FK -> event, unique | one active code per event |
| pin_code | text | 4-6 digit, organizer-visible/regeneratable |
| created_at | timestamp | |

**Check-in access (v1, no team roles)**: organizer generates a per-event PIN (Settings or Event page). Gate staff open the scanner PWA, enter that PIN once to unlock scanning for the event — no staff account/login needed, keeps scope inside the "no team roles in v1" decision (PRD sect. 4 Out of Scope) while still letting non-organizer staff work the gate.

## 5. Core API Endpoints (draft)
- `POST /events` / `GET /events/:id` / `PATCH /events/:id` — event CRUD (organizer auth).
- `POST /events/:id/ticket-types` — create ticket type.
- `POST /orders` — start checkout, return payment intent (or free-confirm directly).
- `POST /webhooks/payment/:gateway` — payment status callback, trigger QR gen + email on success.
- `POST /events/:id/checkin-pin/verify` — verify gate PIN, start scanner session (no staff login).
- `GET /events/:id/checkin-pin` / `POST /events/:id/checkin-pin/regenerate` — organizer view/reset the PIN.
- `GET /attendees/:qr_code/validate?event_day_id=` — check-in scanner lookup, day-scoped for multi-day event.
- `POST /attendees/:qr_code/check-in` — mark checked-in for the given `event_day_id`, idempotent (reject duplicate for that day).
- `GET /events/:id/dashboard` — sales summary, sold vs quota.
- `GET /events/:id/attendees?export=csv` — attendee list + export.
- `POST /orders/:id/refund` — refund/cancel a single order. **Bulk refund deferred post-v1** (see sect. 8).
- `POST /events/:id/promo-codes` / `GET /events/:id/promo-codes` — organizer create/list promo code.
- `POST /events/:id/cancel` — cancel the whole event (status → `cancelled`), triggers refund for every paid order under it. Distinct from single-order refund above.
- `POST /organizer/me/verify` — submit KYC (tax_id + document) for verification; `GET /organizer/me/verify` — check status.
- `GET /admin/reports/sales` — platform-wide GMV, commission earned, sales-over-time.
- `GET /admin/organizers` — list all organizer w/ plan tier, verification status, event count, revenue; search/filter.
- `PATCH /admin/organizers/:id/status` — suspend / reactivate an organizer (blocks their event publish + payout while suspended).
- `PATCH /organizer/:id/verify` — admin-side approve/reject a pending KYC submission (counterpart to organizer's own `POST /organizer/me/verify`).
- `POST /auth/signup` — email, password, name, role (organizer|attendee).
- `POST /auth/login` — email + password.
- `GET/POST /auth/google` — Google OAuth start/callback, both roles.
- `POST /auth/forgot-password` — request reset link by email.
- `POST /auth/reset-password` — token + new password.
- `GET /me/orders` — attendee's own order/ticket history (session-scoped, only orders placed while logged in).
- `GET /orders/:id/invoice` — invoice detail; `?format=pdf` for download.
- `PATCH /organizer/me` — update organizer profile, payout bank account.

## 6. Non-Functional Requirements
- **Payment consistency**: order status must be source-of-truth reconciled with gateway webhook — handle webhook retry/idempotency.
- **QR uniqueness**: QR code must be cryptographically unique per attendee, not guessable (avoid ticket fraud).
- **Check-in duplicate prevention**: check_in write must be atomic (DB constraint or transaction), avoid race on rapid duplicate scan at gate.
- **Scanner offline tolerance**: web-based scanner — flag if API unreachable, queue scan locally, sync when online (stretch, TBD if v1 needs this or online-only ok for v1).
- **CSV export**: must handle event with large attendee count without timeout (stream/paginate).
- **Free tier limit enforcement**: check organizer plan_tier before allow event/ticket creation over cap.
- **Rate limiting**: auth endpoints (login, signup, forgot-password), checkout (`POST /orders`), and promo-code apply all need throttling — brute force / abuse surface.
- **Social share metadata**: Public Event Page needs Open Graph tags (title, banner image, description) so links shared to WhatsApp/Instagram render a proper preview card.
- **Data deletion/retention**: attendee/organizer must be able to request personal data deletion (Indonesia UU PDP compliance) — needs an endpoint + retention policy, not just a UI toggle.
- **Legal pages**: Terms of Service, Privacy Policy, Refund Policy must exist and be linked (footer + checkout) — required both for UU PDP compliance and for Midtrans/Xendit merchant onboarding, which typically asks for these URLs. Content is legal/business-owned, not an engineering deliverable, but the page slots and links are.

## 7. Remaining Technical Decisions
- Password-auth implementation — complete a Worker-compatible secure password-hashing spike; custom authentication is blocked until this passes.
- Account-merge policy — same email registers via Google then later via password (or reverse): merge into one account or block second method? Flag in USER_FLOWS.md Flow 0.
- Check-in offline policy — v1 must show an explicit offline failure and manual fallback; any local queue/sync design needs a separate conflict-resolution spec.
- **PPN handling** — confirm production tax treatment and remitter with finance/tax advisor; sandbox tax remains zero (ADR 0003).
- **KYC provider/flow** — v1 uses manual review per ADR 0002; reassess automated verification after the pilot.
- **Category taxonomy** — confirm the fixed `category` enum list (conference/workshop/festival/meetup/other) with product before launch; adding a category later is a migration, not just a UI change.
- **Timezone display default** — confirm whether to always show local event timezone, or convert to viewer's browser timezone with a "local time" label.

## 8. Explicitly Deferred (documented, not built for v1 — see PRD sect. 9)
- ~~**Admin/platform-owner panel**~~ → **now IN v1** (sect. 4/5 `admin` entity + endpoints). **Dispute-resolution workflow specifically stays deferred** — no ticketing/case system, disputes handled manually outside the product for v1.
- **Bulk refund** (multi-order at once, distinct from whole-event cancel above) — v1 ships single-order refund only. Deferred post-v1.
- **Commission reversal on refund** — sandbox policy reverses platform commission for a fully refunded order (ADR 0003). Production financial treatment still needs explicit finance/business sign-off before launch.
- **Ticket transfer / attendee name change** — common real-world request, not scoped for v1. Attendee is locked to whoever registered.
- **Waitlist when sold out** — no queue/notify-when-available mechanism in v1; sold-out ticket type just shows unavailable.
- **Organizer/event moderation** — no pre-publish content-moderation queue in v1. However, verified KYC is required before any organizer can publish or accept registrations (ADR 0002); trust-and-safety content review remains a later concern.
- **WhatsApp notification channel** — QR ticket + order updates are email-only in v1, despite WhatsApp being the dominant channel in the Indonesia market. Worth prioritizing early post-v1.
