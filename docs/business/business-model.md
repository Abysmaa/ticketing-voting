# Ticketing SaaS Business Model

## Core thesis

This is not a broad consumer event marketplace at launch. It is a B2B2C operating system for Indonesian organizers that need to sell tickets, collect local payments, issue QR tickets, and run gate check-in without spreadsheets, manual transfers, or custom technical work.

The first customer is the organizer. The attendee is the user whose checkout conversion and admission experience determines whether the organizer returns.

## Beachhead customer

Target first:

- Recurring workshop, community, meetup, bootcamp, conference, and campus-event organizers.
- Paid events with roughly 50 to 500 attendees.
- Organizer already has an audience on Instagram, WhatsApp, community groups, or campus channels.
- Organizer needs local payment methods, QRIS/VA/e-wallet support, attendee exports, and day-of-event check-in.

Do not target major music festivals, assigned seating, enterprise procurement, or a public marketplace at MVP. Those segments require settlement, compliance, support, and operational maturity the product does not yet have.

## Customer problem and value proposition

| Customer pain | Product promise |
|---|---|
| Manual bank transfer confirmation | Xendit payment status is webhook-authoritative |
| Spreadsheet attendee lists | searchable attendee list and export |
| QR/check-in chaos at the gate | PIN-gated mobile scanner with duplicate rejection |
| Need a website for every event | reusable public event page and checkout |
| Unclear cash/payout position | visible gross sales, fees, pending payout, and audit record |
| Low trust for a new organizer | verified-KYC publish gate and transparent refund terms |

The value proposition is not "the cheapest ticketing tool." It is "launch and operate a legitimate paid event without building payment, ticket, and gate operations yourself."

## Business model

### Revenue stream 1: transaction commission

The launch model is a commission deducted from successful paid-ticket sales.

- Sandbox/pilot price: **5% platform commission of paid ticket subtotal**.
- Buyer sees the organizer's advertised ticket price; no hidden buyer service fee in pilot.
- Payment gateway cost is recorded separately from platform commission and disclosed to the organizer before publish.
- Free registrations have no platform commission.
- Fully refunded orders reverse the sandbox commission.

This aligns revenue with organizer success and removes subscription friction before the organizer has proven value.

### Revenue stream 2: Pro plan after pilot

Do not sell a paid subscription in the first pilot. Instrument demand first, then launch one simple tier:

| Plan | Price | Commission | Best for |
|---|---:|---:|---|
| Starter | Rp0/month | 5% | occasional/small organizer |
| Pro candidate | Rp299.000/month | 2,5% | organizer with repeat monthly GMV |

At these candidate prices, Pro becomes cheaper around **Rp11.960.000 monthly GMV**. Evidence from the financial model: at Rp5M GMV Starter costs Rp250K while Pro costs Rp424K; at Rp30M GMV Starter costs Rp1,5M while Pro costs Rp1,049M.

Pro is not enabled until the pilot demonstrates repeat organizer usage. It should add operational value, not just a lower fee: higher event/ticket caps, branded domain, priority support, and deeper reports can be validated later.

### Explicitly not monetized in MVP

- No advertising/sponsored event placement.
- No buyer-facing service fee.
- No markup on gateway fees.
- No white-label enterprise setup fee.
- No marketing automation add-on.

These create trust, accounting, or support complexity too early.

## Money flow

```text
Buyer pays advertised ticket price through Xendit
  -> order becomes paid only from verified Xendit webhook
  -> platform records gross sale, gateway cost, platform commission, and organizer payable
  -> event occurs
  -> after event end + 7 days, verified organizer enters manual payout queue
  -> admin transfers payout and records immutable reference
```

The organizer dashboard must always separate:

1. Gross ticket sales.
2. Discounts/refunds.
3. Gateway charges.
4. Platform commission.
5. Pending versus paid-out organizer balance.

Never call gross sales "revenue" in the organizer view. It is GMV; platform revenue is the commission.

## Unit economics measurement

Do not claim profitability from the 5% fee until measured. For every paid order and every organizer event, capture:

```text
platform contribution
= platform commission
- payment gateway charge
- transactional email/QR delivery cost
- refund/chargeback loss
- allocated support and KYC-review cost
```

Pilot gates:

- Gross margin per completed paid order must be positive after gateway and direct delivery cost.
- Refund/chargeback rate must remain below a threshold set before paid production launch.
- Manual KYC review and payout operation must have a documented time cost per organizer.
- If small-ticket events are unprofitable at 5%, use a transparent organizer minimum commission or change the segment; do not hide fees from buyers.

## Go-to-market: supply first, not marketplace first

### Phase A: concierge pilot

Recruit 5 to 10 trusted organizers with a real upcoming event. Founder/onboarding support is manual. The goal is learning the whole flow, not maximizing signups.

Offer pilot terms:

- No subscription.
- 5% commission clearly disclosed.
- KYC before publishing.
- Manual payout H+7 after event.
- Named support channel during setup and event day.

### Phase B: repeat-organizer proof

A pilot succeeds only when an organizer creates a second event without concierge setup. Track whether the organizer reuses the platform, not only whether one event was processed.

### Phase C: narrow referral loop

After repeat usage is proven, add organizer referral: refer a verified organizer who completes their first paid event, then grant a limited commission credit. Do not introduce attendee affiliate links yet.

## North-star and operating metrics

| Layer | Metric | Why it matters |
|---|---|---|
| North star | completed paid admissions | combines sale completion and real event operation |
| Activation | organizer publishes verified event and receives first paid order within 7 days | proves setup-to-value |
| Retention | organizers creating another event within 60 days | validates B2B repeat value |
| Liquidity | checkout-to-paid conversion | exposes payment/UX friction |
| Quality | duplicate-scan rate, scanner failure rate, webhook failure/retry rate | validates day-of-event reliability |
| Trust | refund rate, chargeback rate, KYC rejection rate, payout SLA | protects the marketplace side |
| Economics | net platform contribution per paid order/event | prevents growth that loses money |

## Operating model

- KYC queue: manual admin review before publication.
- Payout queue: weekly manual, earliest H+7 after event conclusion.
- Support: one organizer support channel for pilot; define incident escalation before the first live event.
- Refund: organizer requests within product; platform/admin manages exceptional cases with audit trail.
- Legal: Terms, Privacy, and Refund Policy are launch blockers for paid production.

## Key risks and kill criteria

| Risk | Signal | Response |
|---|---|---|
| No organizer repeat use | fewer than 3 of first 10 organizers create a second event | interview churned organizers before expanding acquisition |
| Low margin on low ticket prices | direct costs absorb most of 5% | introduce transparent organizer minimum fee or focus on higher-value events |
| KYC creates too much friction | high incomplete/rejected KYC before publish | improve onboarding/document requirements, not bypass verification |
| Payment/support burden is too high | frequent pending payment/manual support | inspect Xendit flow and checkout state before adding marketing features |
| Gate failure harms trust | scanner incident at pilot event | prioritize scanner reliability and manual fallback over new dashboard features |

## 90-day proof plan

1. Onboard 5 trusted organizers with real events.
2. Complete at least 3 paid events end-to-end: publish -> Xendit payment -> QR -> check-in -> refund/payout reconciliation.
3. Measure event-level contribution and support time.
4. Interview every organizer and at least five attendees per completed event.
5. Continue only if repeat organizer creation, positive contribution, and operational reliability meet pre-agreed thresholds.

## Decisions that need explicit owner before paid launch

- Final commission/minimum-fee policy after Xendit cost model is known.
- Tax/remitter and invoice treatment from finance/tax advisor.
- Refund/chargeback reserve and dispute process.
- Payout legal/contractual terms.
- Target segment after pilot evidence.