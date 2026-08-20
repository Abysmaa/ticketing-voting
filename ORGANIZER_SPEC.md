# Ticketing SaaS — Organizer Panel Spec v1

Same depth as `ADMIN_SPEC.md`, applied to the Organizer role. Cross-references `PRODUCT_SPEC.md`, `TRD.md`, `USER_FLOWS.md` (Flow 1, 5, 6, 7, 9, 10). Doc only — wireframe status noted per page since most of this role is already built; this doc doubles as a completeness checklist.

## 1. Scope & Access Model
- Full account, email+password or Google (Flow 0) — no guest mode for organizer (unlike Attendee).
- One organizer account = one business/org for v1 (no team roles/multi-tenant — PRD sect. 4 Out of Scope).
- Payout gated on `verification_status = verified` (KYC, TRD sect. 4).

## 2. Page List (summary)

| # | Page | Purpose | Status |
|---|---|---|---|
| 1 | Login / Signup / Forgot / Reset | Shared Flow 0 auth | Built |
| 2 | Dashboard — populated | List own events | Built |
| 2b | Dashboard — **empty state** | First-time organizer, zero events | Built (as a tweak toggle on the Dashboard artboard) |
| 3 | Event Form (create/edit) | Title, date/timezone, venue, category, type, banner | Built |
| 3b | Event Preview | Organizer previews public page before publish (Flow 1) | **Not built** — likely reuses Public Event Page in a preview mode, needs confirming, not a new design |
| 4 | Ticket Type Form | Name, price, quota, sale window, per-day/full-pass | Built |
| 5 | Promo Code Form | Code, discount, scope, max uses | Built |
| 6 | Sales Summary tab | Revenue, sold/quota, per-type breakdown | Built |
| 7 | Attendee List tab | Search/filter/export | Built |
| 8 | Report tab | Sales-over-time, check-in rate, quota fill | Built |
| 9 | Order Detail / Refund modal | Single-order refund | Built |
| 10 | Cancel Event modal | Whole-event cancel, mass refund | Built |
| 11 | Settings | Profile, Payout, KYC, Check-in PIN, Plan & Billing | Built |
| 12 | **Plan Upgrade / Billing comparison** | Free-vs-Paid comparison, upgrade CTA (Flow 6) | Built |

## 3. Page-by-Page Detail (new/gap items only — built pages already detailed in USER_FLOWS.md)

### 3.1 Dashboard — Empty State
- **Content**: no event-list table; instead a centered illustration/placeholder + "Create your first event" headline + the same "+ Create Event" CTA as the populated state.
- **Requirement**: this is Phase 0's own exit criteria ("organizer can sign up, log in, land on empty dashboard") — currently unverified since only the populated wireframe exists. Cheap to add, should not ship without it since it's literally every new organizer's first screen.

### 3.2 Event Preview
- **Content**: same rendering as the public Event Page (Homepage → Public Event Page), but reachable from the Event Form / Ticket Type Form flow before Publish, and probably with a "Draft Preview" banner + no working Buy button (or a disabled one).
- **Requirement**: confirm with team whether this is a literal reuse of the Public Event Page component in a preview mode (recommended — avoids maintaining two templates) vs a distinct page. Not a new visual design either way, just a routing/state decision.

### 3.3 Plan Upgrade / Billing Comparison
- **Content**: side-by-side Free vs Paid comparison (event/attendee cap numbers — still TBD, TRD sect. 7 — vs unlimited; commission rate difference), "Upgrade" CTA, billing form (payment method for the subscription itself, separate from ticket-sale payment processing).
- **Trigger**: (a) organizer hits free-tier cap while creating event/ticket (Flow 6), or (b) clicking "Manage Billing" from Settings.
- **Requirement**: needs the free-tier limit numbers resolved (PRD sect. 7 "Still open") before this page's content is final — currently would need placeholder numbers.

## 4. Cross-Cutting Requirements (not yet explicit elsewhere)
- **Event Form validation**: sale_end must be ≤ event date; ticket quota must be > 0; multi-day event needs ≥ 1 day before Ticket Type step allows per-day scoping (Flow 1 edge case, already documented).
- **Draft vs Published state**: what organizer-editable fields lock once an event has sold ≥ 1 ticket (e.g. can date change after tickets sold? Recommend: date/venue edits should warn-and-notify-attendees rather than silently change — not yet decided, flag as open question).
- **Settings save behavior**: confirm whether Settings sections save individually (per-section Save) or as one page-level Save — current wireframe implies one Save button for the whole page; worth confirming since KYC document upload probably needs its own submit action distinct from Profile/Payout edits.

## 5. Recommended Phasing
- **Must before launch**: Dashboard empty state, Plan Upgrade/Billing page — both were direct gaps in already-committed Phase 0/Phase 5 scope (ROADMAP.md), not new asks. **Both now wireframed.**
- **Should confirm, not necessarily build new**: Event Preview reuse decision.
