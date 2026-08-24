# Ticketing SaaS Threat Model v1

## Assets to protect

- Buyer money and order status.
- Ticket inventory and check-in admission decisions.
- QR/access tokens and session credentials.
- Organizer bank data and KYC documents.
- Admin permissions and audit history.

## High-risk threats and controls

| Threat | Control |
|---|---|
| Last-ticket oversell | Event Durable Object serializes holds/confirmation; expiry releases holds; integration test uses concurrent requests |
| Forged/replayed payment webhook | Verify Xendit signature, unique provider event id, expected amount/currency check, transactionally persist event before side effects |
| Browser fakes payment success | Browser callback is display-only; only webhook can issue ticket |
| Guessable ticket/order URL | random QR secret and signed/hashed guest-access token; no sequential ids exposed as authorization |
| Duplicate gate admission | idempotent Event Durable Object command plus unique check-in database constraint |
| Stolen admin session | short session, mandatory 2FA, role checks, rate limits, audit log |
| KYC/bank disclosure | private R2 object keys, Worker authorization, masked display, no public bucket URL |
| Promo/brute-force abuse | rate-limit promo apply, checkout, login, reset, and PIN verification; add Turnstile after UX validation |
| Queue/Workflow replay | idempotency key on every consumer command; outbox written in the source D1 transaction |
| PII in logs | structured allowlist logs; redact email, bank, token, QR, document keys, and payment secrets |

## Security tests required before pilot

1. Replay the same valid webhook event twice: one order and one ticket only.
2. Submit invalid signature and amount-mismatched webhook: no order state change.
3. Parallel reserve/finalize last ticket: exactly one paid/free-confirmed result.
4. Scan same QR concurrently: exactly one accepted check-in.
5. Fetch another buyer's order with guessed id/no token: denied.
6. Access KYC object as attendee/organizer other than owner: denied.
7. Execute admin payout/KYC route without 2FA: denied and logged safely.

## Explicit non-goals

- No payment-card storage.
- No custom cryptography or browser-issued payment authority.
- No offline scanner sync until a conflict-resolution design is approved.
