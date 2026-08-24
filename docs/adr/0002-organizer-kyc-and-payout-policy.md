# ADR 0002: Gate Public Event Publishing and Payout by KYC

## Status

Accepted as the conservative MVP policy on 2026-08-23.

## Decision

An organizer may sign up and create/edit drafts before verification. The organizer must pass manual KYC before any event can be published publicly, accept registrations, or receive payouts.

## Rationale

Ticketing collects buyer funds and exposes the platform to fraud, chargebacks, fake events, and payout-account abuse. A strict publish gate is simpler and safer than allowing public selling then trying to recover funds after the fact. It also gives admin a single operational queue: submitted KYC -> reviewed -> verified/rejected.

## MVP workflow

1. Organizer signs up and creates an event draft.
2. Organizer submits legal name/business name, contact email, bank-account details, tax/business identifier when applicable, and supporting document.
3. Admin reviews, records an approve/reject reason, and the action is appended to `admin_audit_log`.
4. Only `verified` organizers can publish. Rejected organizers can correct and resubmit.
5. Payout is manual, weekly, no earlier than seven days after the event end date. Every payout requires verified KYC, an immutable payout record, reference number, and admin audit entry.

## Security requirements

- KYC documents and bank data are private R2 objects, served only through a role-checked Worker route.
- Bank account data is masked in normal admin views.
- Admin 2FA is required before KYC and payout routes are enabled.
- Suspension blocks publication, sale, and payout; existing paid orders require an explicit operational decision, never a silent cancellation.

## Review trigger

Revisit when an automated KYC provider or gateway-managed subaccount/split payout is selected.