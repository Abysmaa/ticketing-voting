# Ticketing SaaS — Technical Requirements Document (TRD) v1

Derived from `PRODUCT_SPEC.md` (PRD). This doc turn product scope into build-able tech spec.

## 1. Goal
Answer *how* to build what PRD defines as MVP. Target: single-tenant ticketing platform, web checkout + web-based check-in scanner, Indonesia payment gateway.

## 2. Architecture Overview
- **Web app** (organizer dashboard + attendee checkout) — single web app, server-rendered or SPA + API.
- **API/backend** — REST (or GraphQL, TBD) serving web app + check-in scanner (also web-based, PWA).
- **DB** — relational (Postgres recommended — strong fit for order/payment consistency, relational entity per PRD sect. 6).
- **Async worker/queue** — for QR generation + email send after payment confirm, and payment webhook processing.
- **Payment gateway integration** — unified gateway abstraction layer behind the API; buyer only ever sees payment METHODS (VA/e-wallet/QRIS/card), never a gateway name. v1 launches wired to **one** underlying gateway (Xendit or Midtrans — vendor pick is a business decision, both SDKs cover the needed methods); architecture supports adding the second gateway later without any buyer-facing or API-contract change. Webhook callback drives payment status either way.
- **Check-in scanner** — PWA, browser camera access (`getUserMedia` / barcode-detector API), hit API to validate + mark check-in.

```
[Attendee/Organizer Browser] ---> [Web App / API] ---> [Postgres DB]
                                        |      \
                                        |       -> [Queue/Worker] -> [Email/QR gen]
                                        v
                              [Payment Gateway Webhook]
[Check-in Scanner (PWA)] ---> [API: validate QR] ---> [DB: mark checked-in]
```

## 3. Tech Stack (proposed — confirm with team)
- Backend: Node.js (NestJS/Express) or similar — TBD based on team preference.
- Frontend: React/Next.js — supports SPA dashboard + PWA scanner in one codebase.
- DB: PostgreSQL.
- Queue: Redis + BullMQ (or equivalent) for async job (email, QR gen).
- File/asset storage: S3-compatible (event banner, QR image) — TBD provider.
- Payment: Midtrans and/or Xendit SDK.
- Auth: session/JWT. Both Organizer and Attendee get full account: **email + password** signup/login, plus **Google OAuth** social login. Separate account pools (organizer vs attendee), same auth mechanism/UI pattern. Attendee checkout still offers **guest-checkout** (no login required to buy) alongside "Log In / Sign Up" — logging in at checkout ties the order to the account so it shows in "My Tickets"; a pure guest order is reachable only via the emailed confirmation link, not the dashboard.
- Forgot/reset password: standard email-link token flow (request → emailed link → set new password).

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
| tax_amount | numeric | PPN 11% — v1 assumption: ticket price shown to buyer is **tax-inclusive**, tax broken out on invoice for compliance (confirm w/ finance, sect. 7) |
| total_amount | numeric | subtotal + tax − discount |
| commission_amount | numeric | computed at order time from organizer commission_rate, on **subtotal** (pre-tax) — confirm w/ finance |
| payment_gateway | enum(midtrans, xendit, none) | none = free ticket |
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

## 7. Open Technical Decisions (needs team input)
- Auth provider — build password + Google OAuth in-house (bcrypt/argon2 + Google client) vs managed (Auth0, Clerk, Supabase Auth).
- Account-merge policy — same email registers via Google then later via password (or reverse): merge into one account or block second method? Flag in USER_FLOWS.md Flow 0.
- REST vs GraphQL for API.
- Exact free-tier limits (event count / attendee count cap).
- Exact commission % rate, and whether per-organizer override needed.
- Offline support requirement for check-in scanner.
- Email provider for QR delivery (SES, SendGrid, etc).
- **Which gateway launches v1 first** — Xendit vs Midtrans (business/cost decision; abstraction layer sect. 2 means this doesn't block architecture).
- **PPN handling** — confirm tax-inclusive pricing assumption (sect. 4 Order) and whether platform or organizer is the remitting party, with finance/tax advisor.
- **KYC provider/flow** — manual document review by platform team vs automated verification service (e.g. Privy, Verihubs) for organizer tax_id + document check.
- **Category taxonomy** — confirm the fixed `category` enum list (conference/workshop/festival/meetup/other) with product before launch; adding a category later is a migration, not just a UI change.
- **Timezone display default** — confirm whether to always show local event timezone, or convert to viewer's browser timezone with a "local time" label.

## 8. Explicitly Deferred (documented, not built for v1 — see PRD sect. 9)
- ~~**Admin/platform-owner panel**~~ → **now IN v1** (sect. 4/5 `admin` entity + endpoints). **Dispute-resolution workflow specifically stays deferred** — no ticketing/case system, disputes handled manually outside the product for v1.
- **Bulk refund** (multi-order at once, distinct from whole-event cancel above) — v1 ships single-order refund only. Deferred post-v1.
- **Commission reversal on refund** — default v1 assumption: commission is earned at successful payment and is **not** reversed on refund (platform absorbs no loss on cancelled orders). Needs explicit finance/business sign-off before launch — flag if this assumption is wrong.
- **Ticket transfer / attendee name change** — common real-world request, not scoped for v1. Attendee is locked to whoever registered.
- **Waitlist when sold out** — no queue/notify-when-available mechanism in v1; sold-out ticket type just shows unavailable.
- **Organizer/event moderation** — no review/approval step before an event goes public in v1; any verified-or-not organizer can publish and collect payment immediately. Trust & safety risk to revisit once volume justifies it.
- **WhatsApp notification channel** — QR ticket + order updates are email-only in v1, despite WhatsApp being the dominant channel in the Indonesia market. Worth prioritizing early post-v1.
