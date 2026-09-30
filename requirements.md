# Requirements

## Program scope (governing): Project Siddhi Phase 1

See [references/project-siddhi-phase1-backlog.md](references/project-siddhi-phase1-backlog.md).
- **Manatee, Florida, Personal Home & Dwelling. Renewal processing only (no new-business quoting).** Policies migrate at renewal.
- **First renewal offer: April 2027.** Duck Creek and Siddhi run side by side for 12–16+ months.
- Billing's Phase 1 backlog is **Stages 5–7, features 34–62**, plus feature 70 (flat cancellation when a renewal invoice is unpaid). Each feature has a Day 1 / Confirm / Day 2 / Out-of-scope status.
- Billing rules default to the **current Duck Creek configuration**, validated with the client.
- **Out of scope:** recurring-payment collection (Heartland, feature 56) and queue/task management (FileSphere owns it).

## Current focus: Billing domain

Our team owns **Billing**, and as part of the Portal team we build both the Billing APIs and the Billing UI: internal operations screens **and** the Customer/Agent Portal billing screens (plan §8.1a, §9.2). Policy, Claims and Party belong to other teams: integrate with them through their contracts and don't build their features.

Full spec: [architecture/billing-domain.md](architecture/billing-domain.md)

Business rules: [architecture/billing-rules-from-duckcreek.md](architecture/billing-rules-from-duckcreek.md). Reuse Duck Creek's rules and practices, but don't replicate Duck Creek 1:1.

Key contracts:
- **Policy → Billing**: account setup, premium, fees, instalment plan, effective dates, cancellation and reinstatement.
- **Billing → Policy**: delinquency, payment status (Current / Past Due / Delinquent / Written-off), collections and the non-pay cancellation indicator.

Capabilities in scope: see the plan's §1 table and its feature→phase map (§10.3). **Not in Phase 1:** new-business billing, recurring collection and AutoPay (Heartland, F56), and Billing-owned queues (FileSphere).

## Acceptance criteria

Each phase's exit gate in [plan/billing-implementation-plan.md](plan/billing-implementation-plan.md) §10 is that phase's acceptance criterion.
