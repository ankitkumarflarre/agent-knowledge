# Requirements

## Current focus: Billing domain

Our team owns **Billing**, and as part of the Portal team we build both the Billing APIs and the Billing UI: internal operations screens **and** the Customer/Agent Portal billing screens (plan §8.1a, §9.2). Policy, Claims and Party belong to other teams: integrate with them through their contracts and don't build their features.

Full spec: [architecture/billing-domain.md](architecture/billing-domain.md)

Business rules: [architecture/billing-rules-from-duckcreek.md](architecture/billing-rules-from-duckcreek.md). Reuse Duck Creek's rules and practices, but don't replicate Duck Creek 1:1.

Key contracts:
- **Policy → Billing**: account setup, premium, fees, instalment plan, effective dates, cancellation and reinstatement.
- **Billing → Policy**: delinquency, payment status (Current / Past Due / Delinquent / Written-off), collections and the non-pay cancellation indicator.

Capabilities in scope: invoicing, instalment plans, payment processing (card, ACH/EFT, check, wire, recurring), allocation, suspense, write-off, overpayment and disbursement, void/reverse, commission, delinquency, dunning, collections, persona dashboards (CSR, Underwriter, Agent, Admin, Customer, Management), and the audit trail.

## Acceptance criteria

Each phase's exit gate in [plan/billing-implementation-plan.md](plan/billing-implementation-plan.md) §10 is that phase's acceptance criterion.
