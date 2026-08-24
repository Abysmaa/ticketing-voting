# ADR 0001: Use Xendit as the v1 Payment Gateway

## Status

Accepted on 2026-08-23.

## Decision

Use Xendit as the sole v1 payment gateway behind a small `PaymentGateway` adapter. The product UI exposes payment methods, not the gateway brand.

## Consequences

- Keep the adapter boundary even though there is only one provider in v1; do not implement a speculative second gateway.
- The gateway webhook is authoritative for payment state. Redirect/return pages only poll and display server-side order state.
- Persist gateway event id, payment request id, raw-safe metadata hash, idempotency key, expected amount, and currency before applying a transition.
- Validate webhook signature using a Worker secret and reject mismatched order amount/currency.
- Never collect card data or emulate gateway state in the browser.

## Deferred

A second gateway is not planned until a concrete reliability, method-coverage, or commercial need exists.