# Ticketing SaaS — Product Spec (Draft)

## 1. Vision
General-purpose ticketing/registration platform. Not locked to one event type — same core (event, ticket, order, attendee, check-in) work for conference, workshop, meetup, and multi-day festival.

## 2. Target Users
- Organizer: create event, sell ticket, manage attendee, check-in day-of. *(Full page plan: `ORGANIZER_SPEC.md`)*
- Attendee: browse event, buy ticket / register, get QR ticket, check-in at gate. Includes both Guest (no account) and logged-in Customer state. *(Full page plan: `CUSTOMER_SPEC.md`)*
- Admin (platform owner): oversee organizers, take commission, handle disputes. *(Full page plan: `ADMIN_SPEC.md`)*

## 3. Event Type Support
- **Single-day event**: conference, workshop, meetup. One date, one venue, simple ticket tiers.
- **Multi-day festival**: multi-stage, multi-day pass, add-on item (merch, camping, parking), per-day or full-pass ticket.
- Same data model, differ by config: `event.days[]`, `ticket_type.valid_days[]`, optional `stage`/`session` sub-entity for festival lineup.

## 4. MVP Scope (v1)
### Core
- Event create/edit (title, date(s) w/ timezone, venue, banner, description, category — Conference/Workshop/Festival/Meetup/Other, for browse filter, separate from single-day/multi-day structural type).
- Event cancellation: organizer can cancel the whole event, triggers refund to every attendee at once (distinct from cancelling one order).
- Ticket type: name, price, quota, sale window, per-day or full-pass (for festival).
- Checkout flow: cart, attendee info form, payment.
- Registration: free ticket path (no payment, just form + confirm).
- QR ticket generation per attendee, emailed after purchase.
- Check-in app/scan: validate QR, mark attendee as checked-in, prevent duplicate. Multi-day festival: check-in scoped per day.
- Check-in access: event-scoped PIN (organizer generates/shares), gate staff enter once to unlock scanner — no staff account, matches "no team roles in v1" decision below.

### Organizer Dashboard
- Sales summary: revenue, ticket sold vs quota, per ticket-type breakdown.
- Attendee list: search, filter, export CSV.
- Refund/cancel order (single order — bulk refund deferred, see sect. 9).
- Promo code create/manage (code, discount type/value, scope, max uses).
- Basic report: sales over time chart, check-in rate.
- **Settings**: profile (org/business name, contact email), payout bank account, plan & billing (tier, commission rate).

### Admin Dashboard (Platform Owner)
*Full page-by-page breakdown: see `ADMIN_SPEC.md`.*
- **Platform overview**: total GMV, total commission earned, active organizer count, total live event count, platform-wide sales-over-time chart.
- **Organizer list**: search/filter, per-organizer plan tier, verification status, event count, revenue; suspend/reactivate an organizer.
- **Disputes**: nav placeholder in v1 — no dedicated dispute-resolution workflow built yet (handled manually), see sect. 9.
- Admin account has **no public self-signup** — provisioned internally by the platform team, separate from Organizer/Attendee auth pools.

### Auth (Organizer + Attendee)
- **Email + password** signup/login, plus **Google social login** — both account types.
- **Forgot/reset password** flow (email link).
- Attendee checkout keeps **guest-checkout** option (buy without account, lowest friction) alongside "Log In / Sign Up" — logging in ties the order to the account.

### Attendee Account
- **My Tickets dashboard**: view past/upcoming order (orders placed while logged in), re-access QR ticket, download invoice.
- **Invoice/receipt** per order: itemized ticket, promo discount, total paid, downloadable PDF.

### Out of scope for v1
- Multi-tenant org accounts / team roles / custom branding-subdomain — defer to v2.
- Seat-map/assigned seating.
- Affiliate/referral ticket link.
- Marketing tools (email campaign, discount automation) — only basic promo code in v1.

## 5. Payment
- Gateway: Midtrans and/or Xendit (Indonesia local, cover VA, e-wallet, QRIS, card).
- Platform commission model: **percentage (%) per ticket sold**. Exact rate TBD (business/pricing decision, not technical).
- Organizer plan tier: **Free + Paid tier**. Free tier limited (event/attendee cap, TBD), paid tier unlock higher cap and/or lower commission.
- Payout to organizer: manual or scheduled disbursement (TBD, depend on gateway split-payment support). **Gated on organizer KYC verification** (tax ID + document) — required by gateway before payout enabled.
- Tax: ticket price shown to buyer is **tax-inclusive** (PPN 11%), broken out as a line item on invoice for compliance. Assumption — confirm w/ finance.

## 6. Key Entities (draft data model)
- `Organizer` — account that owns events.
- `Event` — belongs to organizer; has one or more `EventDay`.
- `TicketType` — belongs to event; price, quota, valid day(s), sale window.
- `Order` — attendee purchase/registration; has payment status.
- `Attendee` — one per ticket; has QR code, check-in status.
- `PromoCode` — optional discount, scoped to event or ticket type.
- `CheckIn` — log of scan event (who, when, gate/staff).

## 7. Open Questions — RESOLVED (2026-08-21)
- ~~Pricing/commission model~~ → **Percentage per ticket.** Exact rate still TBD.
- ~~Free vs paid plan tiers~~ → **Free + Paid tier**, v1. Tier limits still TBD.
- ~~Native check-in app vs web scanner~~ → **Web-based scanner (PWA)**, v1.
- ~~Multi-tenant timeline~~ → **Stays v2**, confirmed out of v1 scope.

### Still open (lower priority, resolve during build)
- Exact commission % rate.
- Free tier limit (max event / max attendee count).
- Payout schedule (manual vs automatic disbursement).

## 8. Later / v2+
- Multi-org accounts, team roles, custom branding/subdomain.
- Payment split per organizer (automated payout).
- Seat-map assigned seating.
- Affiliate/referral link tracking.
- Marketing automation (email campaign, abandoned cart).
- White-label option for enterprise organizer.

## 9. Explicitly Deferred (named in scope, not built for v1)
Named elsewhere in this doc but intentionally not built in v1 — documented so it isn't silently forgotten, not a surprise mid-build:
- ~~**Admin/platform-owner panel**~~ → **now IN v1 scope** (sect. 4 Admin Dashboard) — overview + organizer list built; dispute-resolution workflow specifically still deferred (handled manually).
- **Bulk refund** — single-order refund only in v1; bulk (multi-order) refund UI deferred (whole-event cancellation, sect. 4, IS built for v1 and covers the common case).
- **Commission reversal on refund** — default assumption: commission is NOT reversed when an order is refunded. Needs finance/business sign-off before launch (see TRD sect. 8).
- **Ticket transfer / attendee name change** — attendee stays locked to whoever registered.
- **Waitlist when sold out** — no notify-when-available queue.
- **Organizer/event moderation** — no approval step before an event goes public; trust & safety gap to revisit post-launch.
- **WhatsApp notification** — QR ticket + order update stay email-only in v1 despite WA being the dominant local channel.

## 10. Legal & Compliance Notes (not features — ops/legal checklist before launch)
- Terms of Service, Privacy Policy, Refund Policy pages must exist and be linked (footer + checkout) — required for Indonesia UU PDP compliance and typically requested during Midtrans/Xendit merchant onboarding. Content is legal/business-owned, not an engineering build item.
- Data deletion/retention on user request (UU PDP) — needs a real process, not just documented intent.
