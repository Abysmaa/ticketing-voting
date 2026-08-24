# ADR 0003: Use Explicit Sandbox Defaults for Commercial Rules

## Status

Accepted for development and pilot configuration on 2026-08-23. These are not legal, tax, or public commercial commitments.

## Decision

Implement commercial rules through versioned platform configuration, not hard-coded UI constants. Seed the following sandbox defaults:

| Setting | Sandbox default |
|---|---:|
| Currency | IDR |
| Platform commission | 5% of paid ticket subtotal |
| Free plan | 3 concurrently published events; 500 total tickets per event |
| Paid plan | data model exists but no public billing/upgrade flow is enabled |
| Tax | `tax_amount = 0`; invoices state no tax is calculated in sandbox |
| Refund | full order amount; platform commission reverses for fully refunded orders |
| Payout | manual weekly, after event end plus 7 days, only for verified organizers |

## Rationale

The team does not yet have confirmed pricing, tax/remitter, or subscription decisions. Fake final pricing or an invented PPN line on a real invoice would be worse than a clear sandbox setting. The configuration model permits tests and UI development without silently becoming business policy.

## Production gate

A paid production launch is blocked until a business owner and tax/finance advisor approve: tax treatment/remitter, fee display, commission, refund handling, payout timing, and legal policy pages.