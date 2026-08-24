# Ticketing SaaS Route and State Matrix

## Rule

One wireframe scenario equals one valid state. Do not compose empty, loading, success, sold-out, and error content into one screen.

## Public buyer routes

| Route | Primary state | Required variants |
|---|---|---|
| `/` | published events list | loading, no-results, server error, filtered no-results |
| `/e/:slug` | ticket type available | loading, sale-not-started, sale-ended, partial sold-out, all sold-out, cancelled, unpublished, event-ended |
| `/checkout/:reservationId` | valid held cart | expiring soon, expired, price changed, invalid promo, authentication continuation, payment pending, payment failed |
| `/order/:id` | paid/free-confirmed ticket | pending, failed, refunded, cancelled event, signed-link invalid/expired |
| `/tickets` | attendee order history | unauthenticated, empty, loading, server error |

## Organizer routes

| Route | Primary state | Required variants |
|---|---|---|
| `/o` | organizer with events | first-time empty, KYC required, suspended, loading |
| `/o/events/new` | draft creation | validation errors, image upload failure, unsaved changes |
| `/o/events/:id` | event workspace | draft, published, cancelled, KYC-blocked publish, no sales, loading |
| `/o/events/:id/attendees` | paginated table | no attendees, export queued, filter no-results, error |
| `/o/settings` | profile/KYC/payout setup | unverified, pending, rejected, verified, masked bank data |

## Scanner route

| Route | Primary state | Required variants |
|---|---|---|
| `/scan/:eventSlug` | PIN entry | invalid PIN, rate limited, event cancelled, camera permission unavailable |
| `/scan/:eventSlug/session` | camera scanning | accepted, duplicate, invalid, wrong day, cancelled/refunded ticket, network unavailable, manual-entry fallback, session expired |

## Admin routes

| Route | Primary state | Required variants |
|---|---|---|
| `/admin` | KPI overview | 2FA required, no data, data load error |
| `/admin/kyc` | pending review queue | no submissions, document unavailable, approve/reject confirmation |
| `/admin/payouts` | payout queue | no eligible payout, reference validation error, mark-paid confirmation |
| `/admin/audit-log` | activity log | filters no-results, permission denied |

## Acceptance checks

- Every destructive action has a confirmation screen with irreversible consequence copy.
- Every blocked business condition names the corrective action: for example, KYC required -> open KYC submission; reservation expired -> return to event and reselect ticket.
- Payment success is never rendered until `GET /orders/:id` returns server-authoritative `paid` or `free_confirmed`.
- Mobile scanner buttons and status result are usable with one hand in bright/outdoor conditions.
