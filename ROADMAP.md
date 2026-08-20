# Ticketing SaaS — Development Roadmap v1

Build order for MVP, derived from `PRODUCT_SPEC.md` + `TRD.md`. Each phase = shippable increment, dependency-ordered.

## Phase 0 — Foundation (Week 1-2)
- Repo setup, CI/CD skeleton.
- DB schema migration: `organizer`, `event`, `event_day`, `ticket_type` (core tables from TRD sect. 4).
- Organizer auth: email/password signup/login (Google OAuth + forgot-password can land alongside or slightly after — see Phase 4b for full auth build-out).
- Basic organizer dashboard shell (empty state, nav).

**Exit criteria**: organizer can sign up, log in, land on empty dashboard.

## Phase 1 — Event & Ticket Setup (Week 2-3)
- Event create/edit (title, date(s) w/ timezone, venue, banner, description, category).
- Ticket type create/edit (name, price, quota, sale window, per-day/full-pass).
- Event publish/unpublish/**cancel** flow (cancel triggers mass refund — see Phase 2 for refund logic dependency, so full cancel wiring lands once Phase 2 payment exists).

**Exit criteria**: organizer create full event with 1+ ticket type, publish it, view public event page.

## Phase 2 — Checkout & Payment (Week 3-5)
*Depends on: Phase 1 (needs published event + ticket type to sell)*
- Public event page → cart → attendee info form.
- Free registration path (no payment, direct confirm).
- Paid path: single gateway integration (Xendit or Midtrans, TBD sect. 7) behind the abstraction layer (TRD sect. 2) — buyer only sees payment methods, not the gateway name.
- Payment pending/failed status page + retry action.
- Order + commission calc (percentage model, per PRD decision) + tax breakdown (subtotal/tax/total, PPN 11% assumption).
- QR generation + email delivery on order success (async worker).

**Exit criteria**: attendee can buy paid ticket end-to-end and receive QR email; free registration also works; a failed/pending payment shows a clear retry path instead of a dead end.

## Phase 3 — Check-in (Week 5-6)
*Depends on: Phase 2 (needs attendee + QR to scan)*
- Event-scoped check-in PIN: organizer generate/view/regenerate, staff verify to unlock scanner (no staff account).
- Web-based scanner (PWA), camera QR read, day selector for multi-day event.
- `GET /attendees/:qr_code/validate` + `POST check-in` endpoint, day-scoped, duplicate-prevention logic.
- Check-in status view (who's checked in, live count).

**Exit criteria**: non-organizer gate staff can unlock the scanner with the event PIN (no account needed), scan QR, duplicate scan rejected, check-in count updates live, multi-day event checks in per correct day.

## Phase 4 — Organizer Dashboard & Reporting (Week 6-7)
*Depends on: Phase 2 + 3 (needs order + check-in data)*
- Sales summary: revenue, sold vs quota, per ticket-type breakdown.
- Attendee list: search, filter, CSV export.
- Refund/cancel order (single order — bulk deferred, see PRD sect. 9).
- Basic report: sales-over-time chart, check-in rate.
- Organizer Settings: profile, payout bank account, **KYC verification** (tax ID + document upload, status badge).
- Invoice/receipt generation per order (view + PDF download), itemized with tax line.
- Whole-event cancellation (mass refund to every attendee) — wired here since it needs Phase 2 payment/refund plumbing.

**Exit criteria**: organizer can monitor full event health from one dashboard, manage own settings incl. KYC, cancel a whole event with all attendees refunded, and both organizer/attendee can pull a tax-itemized invoice for any order.

## Phase 4b — Auth & Attendee Account (Week 7, pull earlier if blocking Phase 2 checkout)
*Depends on: Phase 0 (base auth needed by both organizer login and attendee checkout)*
- Email/password signup + login, both roles.
- Google OAuth signup + login, both roles.
- Forgot password / reset password flow.
- Guest-checkout vs Log In/Sign Up choice at checkout (Flow 2/3).
- "My Tickets" dashboard: order history, re-access QR, invoice link (accounts only).

**Exit criteria**: organizer and attendee can both sign up/log in via password or Google, reset a forgotten password, and a logged-in attendee sees all their order in "My Tickets".

**Note**: base email/password auth (organizer side) is actually needed as early as Phase 0 for organizer login — this phase 4b entry covers the full build-out including Google OAuth, password reset, and the attendee-facing "My Tickets" dashboard once order data exists (Phase 2).

## Phase 5 — Plan Tier & Promo (Week 7-8)
- Free vs Paid plan tier enforcement (event/attendee cap — limit TBD, see TRD sect. 7).
- Promo code (basic, event/ticket-type scoped).
- Organizer billing/upgrade flow (paid tier).
- Payout disbursement wired to `verification_status = verified` gate (Phase 4 KYC) — unverified organizer accrues revenue but can't withdraw.

**Exit criteria**: free tier capped correctly, upgrade path works, promo code applies discount at checkout, payout blocked until organizer KYC verified.

## Phase 6a — Admin Dashboard (Week 8, parallel-safe with Phase 5/6)
*Scope below is the light version. See `ADMIN_SPEC.md` for the full page-by-page plan (7 "Must" pages vs the 2 listed here) — re-scope this phase once that doc is confirmed.*
*Depends on: Phase 2 (order data) + Phase 4b/KYC (organizer verification status)*
- Admin auth: internal-only account, no public signup (seeded/provisioned by platform team).
- Platform overview: total GMV, commission earned, active organizer count, live event count, sales-over-time chart.
- Organizer list: search/filter, plan tier, verification status, event count, revenue; suspend/reactivate.
- Admin-side KYC approve/reject action.

**Exit criteria**: platform team can log in, see platform-wide sales, list every organizer, suspend one, and approve/reject a pending KYC submission — all without DB access.

## Phase 6 — Hardening & Launch Prep (Week 8-9)
- Load test checkout under concurrent buyers (quota race condition).
- Payment webhook retry/idempotency audit.
- QR fraud check (uniqueness, no guessable pattern).
- Multi-day festival config test (per-day ticket, valid_days logic).
- Error monitoring, basic analytics.
- **Compliance/ops checklist**: rate limiting live on auth/checkout/promo-apply, Open Graph tags on public event page, Terms of Service/Privacy Policy/Refund Policy pages published and linked, data deletion request process exists.

**Exit criteria**: platform ready for first real event, single-day and multi-day both tested, compliance checklist cleared.

## Dependency Chain Summary
```
Phase 0 (Foundation)
   └─> Phase 1 (Event/Ticket)
          └─> Phase 2 (Checkout/Payment)
                 ├─> Phase 3 (Check-in)
                 ├─> Phase 4 (Dashboard/Report/Settings/Invoice)  [needs Phase 3 data too]
                 └─> Phase 4b (Attendee Account)
                        ├─> Phase 5 (Plan Tier/Promo)
                        ├─> Phase 6a (Admin Dashboard)  [needs Phase 4 KYC status]
                        └─> Phase 6 (Hardening/Launch)
```

## Explicit Non-Goals (do not build in this roadmap — see PRD sect. 4 "Out of scope" and sect. 9 "Explicitly Deferred")
- Multi-tenant org accounts / team roles.
- Seat-map/assigned seating.
- Affiliate/referral link tracking.
- Marketing automation (email campaign, abandoned cart).
- Native mobile check-in app (web scanner only, per resolved open question).
- Admin dispute-resolution workflow (Admin dashboard itself IS in v1, see Phase 6a — just no case/ticketing system for disputes).
- Bulk refund (single-order refund only in v1; whole-event cancellation IS in v1, see Phase 4).
- Ticket transfer / attendee name change.
- Waitlist when sold out.
- Organizer/event moderation before public listing.
- WhatsApp notification channel (email-only in v1).

## Notes
- Timeline (Week N) = rough estimate, not committed — adjust once team size/velocity known.
- Exact commission %, free-tier limit, payout schedule still TBD (TRD sect. 7) — must lock before Phase 2/5 start.
