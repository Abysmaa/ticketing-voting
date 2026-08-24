# ADR 0004: Use a Provider Adapter with Resend for v1 Transactional Email

## Status

Accepted on 2026-08-23.

## Decision

Use a small `TransactionalEmail` adapter and start v1 with Resend for QR-ticket, order-confirmation, password-reset, and operational emails. Requests are emitted only from an idempotent Queue consumer after the D1 business transition commits.

## Rationale

Cloudflare Email Service can send transactional email directly from Workers, but Cloudflare documents Email Sending as Beta. It is valuable for future consolidation and sandbox evaluation, but a ticket delivery channel should not depend solely on a beta product for the initial production pilot. The email adapter keeps the Worker deployment Cloudflare-first while allowing a proven delivery provider and a later Cloudflare Email Service migration/evaluation without touching order logic.

## Requirements

- Verify the sending domain and configure SPF, DKIM, and DMARC before production.
- Store delivery request ids and provider webhook outcomes without logging QR or signed-link secrets.
- Include text and HTML variants.
- Retry transient failure with bounded backoff; suppress permanent bounces.
- Ticket email contains a signed non-guessable order-access link, never a raw sequential order id.

## Review trigger

Evaluate Cloudflare Email Service again when its production suitability and required delivery/observability features have been demonstrated for the expected ticket volume.