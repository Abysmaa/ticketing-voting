# Ticketing SaaS — Customer & Guest Spec v1

Same depth as `ADMIN_SPEC.md`, applied to the buyer side. Covers **two access states of one PRD role (Attendee)**, not two accounts:
- **Guest** — no account, browses and can complete guest-checkout (PRD sect. 4, TRD sect. 3 Auth). Reaches order/ticket/invoice only via the emailed link, never a dashboard.
- **Customer** — logged in (email+password or Google), same buying flow but ties orders to an account, gets "My Tickets" and an editable profile.

Cross-references `USER_FLOWS.md` Flow 2, 3, 8. Doc only — most of this role is already built; this doc is a completeness checklist plus the parts that were never fully pinned down.

## 1. Page List (summary)

| # | Page | Guest | Customer | Status |
|---|---|---|---|---|
| 1 | Login / Signup / Forgot / Reset | — (n/a) | Yes | Built |
| 2 | Homepage (browse/search/category) | Yes | Yes | Built |
| 2b | Homepage — **no results state** | Yes | Yes | Built (tweak toggle on Homepage artboard) |
| 3 | Public Event Page | Yes | Yes | Built |
| 3b | Public Event Page — **sold-out state** | Yes | Yes | Built (tweak toggle on Public Event Page artboard) |
| 4 | Cart / Attendee Info Form (guest-or-login choice) | Yes | Yes | Built |
| 5 | Payment Method | Yes | Yes | Built |
| 6 | Payment Pending | Yes | Yes | Built |
| 7 | Payment Failed | Yes | Yes | Built |
| 8 | Order Confirmation | Yes (via emailed link) | Yes (in-session or emailed link) | Built, but **tokenized-link access not yet a stated requirement** — see 3.1 |
| 9 | My Tickets Dashboard | No | Yes | Built |
| 10 | Invoice / Receipt | Yes (via emailed link) | Yes | Built |
| 11 | **Account Settings** (profile/password) | No | Yes | Built |

## 2. Page-by-Page Detail (new/gap items only)

### 2.1 Order Confirmation & Invoice — Guest Access via Emailed Link
- **Requirement (not yet stated anywhere)**: a Guest has no session, so the confirmation email must link to the Order Confirmation / Invoice page via a **signed, tokenized URL** (order id + a non-guessable token, not just the order id alone — otherwise anyone who guesses/enumerates an order id could view another buyer's ticket/PII). This is a real security requirement missing from `TRD.md`'s NFR list (sect. 6) — recommend adding: `GET /orders/:id?token=` pattern, token generated at order-confirm time, embedded in the confirmation email link only.

### 2.2 Guest Order → Account "Claim"
- `USER_FLOWS.md` Flow 8 mentions, parenthetically, "a guest order later claimed via matching email" without ever defining the mechanism. Two real options:
  - **(a) Automatic**: on signup/login, any past guest order matching that email auto-links into "My Tickets" — zero extra UI, but means anyone who later creates an account with an email that placed a guest order inherits that order (acceptable since it's already gated by email ownership via the login flow itself — email/password or Google both prove the buyer controls that inbox).
  - **(b) Manual claim**: a "Claim this order" button/flow, more explicit, more UI to build.
- **Recommendation**: (a), since it needs zero new screen and email ownership is already the trust boundary the auth system relies on. Flag for confirmation, then TRD sect. 5 should gain a line noting `GET /me/orders` includes guest orders matching the logged-in email, not only orders placed while logged in — this **contradicts the current wording** in `TRD.md` sect. 3 Auth ("a pure guest order is reachable only via the emailed confirmation link, not the dashboard") and needs reconciling once decided.

### 2.3 Public Event Page — Sold-Out State
- **Content**: ticket type row shows "Sold Out" instead of the quantity stepper; Buy button at the bottom disabled if every ticket type is sold out. No waitlist (explicitly deferred, `PRODUCT_SPEC.md` sect. 9).
- **Requirement**: this state exists today only implicitly (quota=0 case) — worth a one-line explicit note in `USER_FLOWS.md` Flow 2 since it's a real, common state (popular events sell out), not an edge case.

### 2.4 Homepage — No Results State
- **Content**: empty-state illustration + "No event found" + a "Clear filters" action, when search/category filter returns zero events.
- **Requirement**: cheap, standard empty-state pattern — same category of gap as the Organizer Dashboard empty state in `ORGANIZER_SPEC.md`.

### 2.5 Account Settings (Customer)
- **Content**: name, email (change requires re-verification — flag as requirement), password change (old + new), linked-Google-account indicator, "Delete my account" (ties to the UU PDP data-deletion requirement already noted in `PRODUCT_SPEC.md` sect. 10 — this is the page that action lives on).
- **Requirement**: currently the Customer role has **no page to edit their own profile or password** at all — "My Tickets" only shows order history. This is a real gap, not a nice-to-have: a logged-in user with no account settings page is unusual enough to look unfinished, and it's also where the data-deletion requirement (sect. 10 of `PRODUCT_SPEC.md`) needs to live.

## 3. Cross-Cutting Requirements
- **Guest vs Customer parity**: every buyer-facing page (Homepage through Payment) must render identically for Guest and Customer except the nav (Login/Signup link vs account icon) and the Cart step's guest-or-login choice. No separate template needed.
- **Security**: tokenized order-link access (2.1) is the one real gap — everything else in this doc is completeness, that one is a legitimate security hole if shipped as-is (guessable order URLs).
- **Email verification**: signup flow (Flow 0) doesn't currently mention email verification (click-to-confirm) — worth deciding whether unverified emails can complete purchases, or whether verification is required before an account is "real." Flag as open decision, not resolved here.

## 4. Recommended Phasing
- **Must before launch**: tokenized order-link access (2.1) — **still not built, backend/TRD item, not a wireframe fix** — security gap, not cosmetic. Account Settings (2.5) — **now wireframed**.
- **Should**: Sold-out state (2.3), Homepage no-results state (2.4) — **now wireframed**.
- **Needs a decision, not a build**: guest-order claim mechanism (2.2), email verification requirement.
