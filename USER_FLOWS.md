# Ticketing SaaS — User Flows v1

Screen-by-screen flow for main journey, per PRD roles (Organizer, Attendee) + TRD API/data model. Use to derive wireframe before UI build.

*Deeper page-by-page requirement breakdown, gap/status checklist per role: `ORGANIZER_SPEC.md` (Organizer), `CUSTOMER_SPEC.md` (Attendee — both Guest and logged-in Customer state), `ADMIN_SPEC.md` (Admin).*

## Flow 0 — Auth: Login / Signup / Forgot Password
Shared pattern, used by both Organizer and Attendee (separate account pools, same UI pattern).
```
[Login Screen]
   -> [Email + Password fields] -> [Log In btn]
   -> or [Continue with Google btn] -> [Google OAuth consent] -> [Callback, session started]
   -> [Don't have account? Sign Up link] -> [Signup Screen]
   -> [Forgot password? link] -> [Forgot Password Screen]

[Signup Screen]
   -> [Name, Email, Password, Confirm Password fields] -> [Sign Up btn]
   -> or [Continue with Google btn]
   -> [Account created, session started] -> redirect to role home (Organizer Dashboard / Homepage)

[Forgot Password Screen]
   -> [Enter Email] -> [Send Reset Link btn] -> [Check-your-email notice]
   -> [Click link in email] -> [Reset Password Screen: New Password, Confirm]
   -> [Reset Password btn] -> [Password updated] -> [Login Screen]
```
**Edge case**: reset link expired/reused — show clear error, offer resend.
**Edge case**: Google account email already registered via password — merge to same account (don't create duplicate), or show "already registered, log in with password" — TBD, flag for team.

## Flow 1 — Organizer: Create & Publish Event
```
[Login/Signup] -> [Dashboard (empty state)] -> [Create Event btn]
   -> [Event Form: title, date(s), venue, banner, description]
   -> [Save as Draft] -> [Add Ticket Type form: name, price, quota, sale window, per-day/full-pass]
   -> (repeat add ticket type as needed)
   -> [Preview public event page]
   -> [Publish btn] -> [Event Live] -> back to [Dashboard: event list]
```
**Edge case**: multi-day festival — Event Form must collect `event_day[]` first, so Ticket Type form can offer per-day/full-pass choice tied to those days.

## Flow 2 — Attendee: Browse & Buy Ticket (Paid)
```
[Homepage: browse/search event, filter by category — Conference/Workshop/Festival/Meetup/Other, independent of single-day/multi-day structural type] -> [Public Event Page]
   -> [Select Ticket Type + Qty] -> [Add to Cart]
   -> [Cart Review] -> [Continue as Guest] or [Log In / Sign Up] (Flow 0, optional — saves order to account)
   -> [Attendee Info Form: name, email per attendee]
   -> [Promo Code input (optional)] -> [Payment Method select: VA/e-wallet/QRIS/card — gateway behind the scenes, buyer never sees "Midtrans" or "Xendit"]
   -> [Redirect to gateway / inline widget] -> [Payment Result]
   -> (success) -> [Order Confirmation page] -> [QR emailed async]
   -> (pending, e.g. unpaid VA) -> [Payment Pending screen: VA number/QR shown again, Refresh Status, Cancel Order]
   -> (failed) -> [Payment Failed screen: reason, Retry Payment btn -> back to Payment Method]
```
**Edge case**: quota race — if ticket sell out between "Add to Cart" and payment confirm, must reject at order-creation step (server-side quota check), show clear error.

## Flow 3 — Attendee: Free Registration
```
[Homepage or direct link] -> [Public Event Page] -> [Select Free Ticket Type] -> [Add to Cart]
   -> [Attendee Info Form] -> [Confirm Registration btn] (no payment step)
   -> [Order Confirmation page] -> [QR emailed async]
```

## Flow 4 — Staff: Check-in at Gate
```
[Open Check-in Scanner (PWA, mobile browser)] -> [Enter Event PIN screen] -> [Verify] -> [Unlocked]
   -> (if multi-day event) [Select Day: Day 1 / Day 2 / ...]
   -> [Camera permission grant]
   -> [Scan QR] -> [API validate for selected day] ->
        (valid, not yet checked-in this day) -> [Green: show attendee name] -> [Mark checked-in]
        (already checked-in this day) -> [Red/Yellow: "Already checked in at HH:MM"]
        (invalid/not found, or wrong day) -> [Red: "Invalid ticket"]
   -> [Ready for next scan] (loop)
```
**Edge case**: two staff scan same QR near-simultaneously — API check-in write must be atomic, second scan gets "already checked-in" not duplicate success.
**Note**: no staff account/login — PIN is event-scoped and shared by the organizer to whoever works the gate (matches "no team roles in v1" decision). Organizer views/regenerates the PIN from Settings or the Event page.

## Flow 5 — Organizer: Monitor Event (Dashboard)
```
[Dashboard] -> [Select Event] -> [Sales Summary tab: revenue, sold/quota, per-type breakdown]
   -> [Attendee List tab: search/filter, Export CSV btn]
   -> [Report tab: sales-over-time chart, check-in rate]
   -> [Order detail] -> [Refund/Cancel btn] -> [Confirm] -> [Status updated]
```

## Flow 6 — Organizer: Plan Tier / Upgrade
```
[Dashboard] -> [hits Free tier cap when creating event/ticket] -> [Upgrade prompt modal]
   -> [Plan comparison: Free vs Paid] -> [Upgrade btn] -> [Billing form] -> [Confirm]
   -> [Plan tier updated] -> [cap lifted, continue action]
```

## Flow 7 — Organizer: Settings
```
[Dashboard] -> [Settings nav] -> [Profile section: edit org name/contact email]
   -> [Payout Bank Account section: bank, account number, holder name]
   -> [Verification (KYC) section: Tax ID (NPWP), business document upload -> status: Unverified/Pending/Verified]
   -> [Check-in Access section: per-event PIN, Regenerate]
   -> [Plan & Billing section: current tier, commission rate, Manage Billing btn]
   -> [Save] -> [Confirmation toast]
```
**Note**: payout stays disabled until Verification status = Verified — revenue still accrues, withdrawal blocked.

## Flow 10 — Organizer: Cancel Whole Event
```
[Event page] -> [Cancel Event btn] -> [Confirm modal: "This refunds all N attendee, cannot be undone"]
   -> [Confirm] -> [Event status -> Cancelled] -> [Refund triggered for every paid order]
   -> [Attendee notified by email] -> [Event no longer bookable, shows "Cancelled" on public page]
```
**Distinct from** Flow 5's single-order refund — this cancels the entire event, not one attendee's order.

## Flow 8 — Attendee: My Tickets & Invoice
```
[Homepage] -> [My Tickets nav] -> (not logged in) -> [Login/Signup, Flow 0]
   -> [My Tickets Dashboard: order list, linked to account email]
   -> [View QR btn] -> [QR shown again (same as Order Confirmation)]
   -> [Invoice btn] -> [Invoice page: itemized order, total paid] -> [Download PDF]
```
**Note**: only orders placed while logged in (or a guest order later claimed via matching email) show here — a guest order at an email never logged in stays accessible only via the confirmation email link, not the dashboard.

## Flow 9 — Organizer: Create Promo Code
```
[Ticket Types tab] -> [Promo Codes section] -> [+ New Promo Code]
   -> [Form: Code, Discount Type (% or flat), Discount Value, Scope (whole event or specific ticket type), Max Uses, Validity Window]
   -> [Save] -> [Promo code active, appears in list]
```

## Flow 11 — Admin: Platform Overview & Organizer Management
*Light version below. Full page list (KYC queue, payout queue, audit log, events/orders oversight, disputes log) in `ADMIN_SPEC.md`.*
```
[Admin Login (internal account, no public signup)] -> [Admin Dashboard: platform GMV, commission earned, active organizer count, live event count, sales-over-time chart]
   -> [Organizers nav] -> [Organizer List: search/filter, plan tier, verification status, event count, revenue]
   -> [Select organizer] -> [Approve/Reject KYC] or [Suspend/Reactivate]
```
**Note**: Disputes nav exists as placeholder only — no case-management workflow in v1, handled manually.

## Screens Needed (summary, for wireframe checklist)
| Screen | Role | Flow |
|---|---|---|
| Login (email/password + Google) | Organizer, Attendee | 0 |
| Signup (email/password + Google) | Organizer, Attendee | 0 |
| Forgot Password | Organizer, Attendee | 0 |
| Reset Password | Organizer, Attendee | 0 |
| Dashboard (event list) | Organizer | 1, 5, 6 |
| Event Form (create/edit) | Organizer | 1 |
| Ticket Type Form | Organizer | 1 |
| Homepage (public event browse/search) | Attendee | 2, 3 |
| Public Event Page | Attendee | 2, 3 |
| Cart / Attendee Info Form (w/ guest-or-login choice) | Attendee | 2, 3 |
| Payment Method / Gateway redirect | Attendee | 2 |
| Payment Pending | Attendee | 2 |
| Payment Failed | Attendee | 2 |
| Order Confirmation | Attendee | 2, 3 |
| Check-in PIN Entry | Staff | 4 |
| Check-in Scanner (PWA, w/ day selector) | Staff | 4 |
| Sales Summary tab | Organizer | 5 |
| Attendee List tab | Organizer | 5 |
| Report tab | Organizer | 5 |
| Order Detail / Refund | Organizer | 5 |
| Plan Upgrade modal/billing | Organizer | 6 |
| Settings (profile/payout/billing/KYC, incl. check-in PIN) | Organizer | 7 |
| My Tickets Dashboard | Attendee | 8 |
| Invoice / Receipt (w/ tax line) | Attendee, Organizer | 8 |
| Promo Code Form | Organizer | 9 |
| Cancel Event confirm | Organizer | 10 |
| Admin Dashboard (platform overview + sales report) | Admin | 11 |
| Admin Organizer List (manage/suspend/KYC approve) | Admin | 11 |

## Open Item
- Wireframe/visual design not covered here — this doc = logic flow only. Next step: low-fi wireframe per screen (Figma or `design` skill) once flow confirmed.
