# Reference: Insurance Core Platform Architecture ("Flarre Core Platform")

| | |
|---|---|
| Original | [insurance-core-architecture.html](insurance-core-architecture.html) (copy of `insurance_core_architecture 4.html`; open it in a browser) |
| Received | 2026-09-29 |
| Status | Reference architecture, the current target design |
| Billing detail | Extracted to [../architecture/billing-domain.md](../architecture/billing-domain.md) |

## What the file contains

The file is an interactive page. It has a left menu, but only these pages have real content:

| Menu | Status |
|---|---|
| Architecture Diagram | Built: domain map plus the principal dependencies |
| User Interface → Policy | Built: lifecycle, personas, lapse/OOS rules, audit trail |
| User Interface → **Billing** | Built: see [billing-domain.md](../architecture/billing-domain.md) |
| Forms → Policy | Built: the shared **Forms Engine**, used by every module including Billing |
| Enterprise Data and Downstream → Policy | Built: real-time CDC and event flow to ODS, Lake and reporting |
| All other menus: Claims, Portal, Rating, Integration, User Mgmt, Payment Mgmt, and the Billing pages under Forms, Rating, Integration, User Mgmt, Payment Mgmt and Ent. Data | **Placeholders ("Coming soon")** |

## Domain map (4 layers)

1. **Experience and distribution**: Customer/Agent Portal (quote, service, payments, FNOL, claim status, documents) · Partner & Producer (brokers, MGAs, aggregators) · Internal Operations (underwriting, billing, claims and service workbenches).
2. **Insurance core domains**
   - Policy Administration → Product, Rating & UW
   - **Billing** (accounts, invoices, instalments, receivables, collections, refunds) → **Payment Hub** (cards, ACH, gateway, tokenisation, reconciliation)
   - Claims → Claims Ecosystem
   - Party / Customer 360 → Identity & Matching
   - Documents (forms, notices, policies, invoices, claim packs)
3. **Integration and shared platform**: API Gateway/BFF · Event Bus · Workflow & Rules (tasks, approvals, orchestration, SLA) · Security & Observability (IAM, audit, secrets, logs, metrics, traces).
4. **Enterprise data and downstream**: Operational Data Store · Data Lake/Analytics · MDM/Reference Data · GL/Finance/Reinsurance.

## Principal dependencies (verbatim intent)

| Edge | Carries |
|---|---|
| Policy → Billing | Account setup, premium, fees, instalment plan, effective dates, cancellation and reinstatement |
| **Billing → Policy** | Delinquency, payment status, collections and non-payment cancellation indicators |
| Policy → Claims | Policy status, insured risks, coverage, limits, deductibles, endorsements at date of loss |
| Party → All domains | Stable party identity, addresses, communication preferences, roles, relationships |
| Claims → Finance | Reserves, claim payments, recoveries, deductibles, downstream postings |
| Portal → Core | API-led journeys; **no direct access to domain databases** |

## Architecture guardrails (apply to every domain)

- Each domain owns its transactional data, and events are versioned.
- Use **APIs for commands and validation**, and **events for propagation**.
- Carry correlation, policy, account, claim and party IDs end to end.
- Build in idempotency, retries, dead letters and reconciliation.
- Apply least privilege, encryption, consent and audit to all PII.

## Cross-cutting pieces that Billing reuses

- **Forms Engine (shared):** template (DOCX/PDF) → merge fields → data collection from Policy, Billing, Claims and Party → rendering (with an optional external API call, e.g. e-signature) → output. Every output is stored in the DB, emailed, uploaded to a shared folder for print, and indexed in the DMS (versioning, retention, search, role-based access, retrieval logging). Billing's documents through this engine are invoices, dunning letters and commission statements.
- **Downstream data:** Billing publishes invoice, payment, delinquency and commission events through CDC/event stream. Stream processing then enriches them and correlates them with party/policy, then they flow to ODS, Lake/Warehouse and the semantic layer. Management dashboards combine Policy, Billing and Claims, joined on policy, account, claim and party IDs.
- **Policy lifecycle hooks that affect Billing:** bind, issue, endorsement, renewal, cancellation (insured, company or non-pay), notice of cancellation with cure period, reinstatement, and **lapse** (a renewal not accepted or premium not received by expiry leads to Lapsed status and a grace window). **Out-of-sequence** endorsements re-sequence and re-rate the downstream renewal and billing transactions.
