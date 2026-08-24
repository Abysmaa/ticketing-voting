# Ticketing SaaS Wireframe Audit

## Verdict

The board is a useful coverage map, not a build-ready UX specification. It maps the intended roles and most primary screens, but the flow is not yet coherent enough to implement directly. The visible artifacts mix mutually exclusive states, contain unresolved template placeholders, and omit critical state/error/operational behavior.

## What is already covered

- Auth: login, signup, forgot/reset password.
- Organizer: event creation, ticket types, sales, attendees, reports, order refund, cancellation, settings, plans.
- Attendee: browse, event detail, cart, payment result, confirmation, invoice, account.
- Gate staff: PIN entry and QR scanner.
- Admin: dashboard, organizer, KYC, payouts, audit log.

This is good scope coverage for MVP discovery. It should not be treated as evidence that the flows are validated.

## Verified defects in the rendered wireframes

### State collisions

The same artboard renders incompatible states together:

- Homepage shows an event card and the `No event found` empty state at once.
- Public Event Page shows a sold-out ticket, a quantity stepper, an enabled `Buy Ticket` button, and a disabled `Sold Out` button together.
- Organizer dashboard shows both an existing event row and the first-time `No events yet` state.

Each state must become a separate named scenario. A state variant should never be displayed in the default happy-path artboard.

### Data contract is missing

Homepage, public event, and organizer dashboard still expose placeholders such as `{{ev.title}}`, `{{t.name}}`, and `{{ev.status}}`. Before visual implementation, define a typed view model for every screen, including loading, empty, error, unauthorized, sold-out, cancelled, and payment-pending variants.

### Checkout is not launch-ready

The cart has attendee details and a promo input, but is missing:

- Buyer contact versus per-attendee contact semantics.
- Ticket line-item breakdown, discount, tax/PPN, platform fee if any, and final payable amount.
- Explicit terms, refund-policy, and privacy-policy consent.
- Promo validation feedback and applied-discount removal.
- Session expiry and inventory-reservation countdown.
- Clear distinction between free registration and paid checkout.
- A safe back/edit path when payment is pending or fails.

### Public event page is too thin

It needs organizer identity, event status, timezone policy, ticket sale window, ticket terms, cancellation/refund policy, accessibility/contact information, map/venue detail, social share data, and an explicit cancelled/unpublished state. `Sold Out` must disable quantity controls and all checkout CTAs for that ticket type.

### Scanner is not operationally complete

The scanner has a good minimal shape: prominent status, selected day, success feedback, and live count. It still needs camera permission/error UI, manual code entry fallback, offline/network failure behavior, duplicate/wrong-day/invalid result variants, scanner-session expiry, visible event/PIN context, sound/haptic option, and a way to change event or sign out of the gate session. A gate device cannot silently queue scans without a verified conflict strategy.

### Admin is under-specified for financial risk

Admin actions involving KYC, suspension, payouts, and refunds must require reason/confirmation and create visible audit evidence. Admin 2FA is a requirement, not a nice-to-have, because the role accesses KYC and payout data. Sensitive payout and tax data must be masked by default.

## Required flow model before UI implementation

### Buyer flow

1. Browse/search published events.
2. View event and choose currently on-sale ticket quantity.
3. Create a short-lived server-side reservation before collecting payment.
4. Enter buyer and attendee details, consent to policies, and review totals.
5. Choose guest checkout or account sign-in without losing the reservation.
6. Start payment or confirm free registration.
7. Show pending payment until the server receives an authoritative gateway webhook.
8. Issue QR ticket only after `paid` or `free_confirmed`; email a signed access link.
9. Permit the buyer to revisit an order by authenticated account or signed email link.

### Organizer flow

1. Sign up and complete an onboarding checklist.
2. Create draft -> add dates and ticket types -> preview -> publish.
3. Monitor sales, attendees, check-in, refund, and event cancellation from an event context.
4. Submit KYC and payout data; payout remains blocked until verification is complete.

### Gate flow

1. Staff opens a dedicated scanner route.
2. Staff enters event-scoped PIN and chooses a day when required.
3. Scanner validates one QR at a time and returns a dominant success, duplicate, invalid, or offline state.
4. Check-in is idempotent; the server, not the browser, is authoritative.

## Acceptance gate

Do not start high-fidelity UI or backend endpoints until every core screen has a single primary scenario and named variants for empty, loading, validation failure, authorization failure, business rule failure, and irreversible actions.