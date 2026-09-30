# Billing Domain — Architecture & Requirements

> **Owner:** the Billing team, which is the user's team. We are part of the Portal team, so we also build the Customer/Agent Portal billing APIs and UI. Other teams own Policy, Claims and Party.
> **Source:** [../references/insurance-core-architecture.md](../references/insurance-core-architecture.md). This covers the "Architecture Diagram" page and the "User Interface → Billing" page.
> **Detailed business rules:** [billing-rules-from-duckcreek.md](billing-rules-from-duckcreek.md). Rules carried over from Duck Creek, including pay plans, the PCN/cancellation model, fees and collections.
> Statements marked *(inferred)* come from reading the diagram, not from explicit text. Confirm them before relying on them.

## 1. Scope

Billing owns **accounts, invoices, instalments, receivables, collections and refunds**. Its sub-domain is the **Payment Hub** (cards, ACH, gateway, tokenisation, reconciliation). Billing also does **commission calculation** and produces commission statements.

Billing does **not** own:
- Premium calculation. Rating sits in Policy / Product, Rating & UW.
- The cancellation decision. Billing only sends an *indicator*, and Policy cancels.
- Party identity. Party / Customer 360 owns it.
- Document rendering. The shared Forms Engine does that.

## 2. Dependencies

### Inbound: Policy → Billing *(primary contract)*
Billing receives: **account setup, premium, fees, instalment plan, effective dates, cancellation and reinstatement**. A premium receivable is raised on **bind, issue and endorsement**. Renewal, OOS re-rating, cancellation and reinstatement events also change billing.

### Outbound: Billing → Policy *(primary contract)*
Billing sends: **delinquency, payment status, collections and non-payment cancellation indicators**.
- Payment status values: `Current` / `Past Due` / `Delinquent` / `Written-off`. Billing syncs the status to Policy **on every change**.
- The non-pay cancellation indicator is raised when delinquency stays unresolved. It is **cleared** if payment arrives before the cancellation effective date, and that halts the cancellation.

### Other dependencies (from the diagram)

| Direction | Counterpart | Purpose |
|---|---|---|
| Portal → Billing | Customer/Agent Portal | Payments, invoices and billing views. Must go through the API Gateway/BFF and never read the Billing DB directly |
| Internal Ops → Billing | Billing workbench | CSR and Admin operations |
| Billing → Party | Party / Customer 360 | Payer/insured identity, addresses, communication preferences |
| Billing → Payment Hub | Gateway, ACH, cards | Authorisation, capture, tokenisation, settlement, reconciliation |
| Billing → Documents / Forms Engine | Shared engine | Invoice, schedule, receipt, dunning, commission and refund documents |
| Billing ↔ Shared platform | API Gateway/BFF, Event Bus, Workflow & Rules, Security & Observability | Commands through APIs, propagation through events, approvals and SLAs through workflow, audit through IAM |
| Billing → Downstream | ODS, Data Lake/Analytics, **GL/Finance** | CDC/events for invoices, payments, delinquency and commissions. GL posting of premium, cash and commission entries |

## 3. Billing flow

**Core path**
1. **Premium Receivable**: premium due, received from Policy on bind, issue or endorsement.
2. **Generate Invoice**: covers premium, fees and taxes. Produces the 📄 *Invoice packet*.
3. **Create Instalments**: payment plan split into a monthly, quarterly or EFT schedule.
4. 🔗 Payment gateway / EFT-ACH processor / card network.
5. **Process Payment**: cash, card, ACH/EFT, check or wire.
6. **Allocate Payment**: applied to the invoice or instalment according to the allocation rules below.

**Exception branches off Allocate Payment**
- **Suspense**: the payment is unmatched or unidentified and is held until it can be matched.
- **Write-off**: a small or uncollectable balance above the tolerance threshold, which needs approval.
- **Overpayment → Disbursement**: a credit balance is found; after approval it is refunded to the payer.
- **Void / Reverse**: NSF, chargeback or error.

**Commission**: commission for the agent or broker is computed **on earned premium** and produces the 📄 *Commission statement*.

**Recurring payments and non-pay path**
- **Recurring EFT**: bank draft on the due date, retried automatically on failure.
- **Recurring card**: card charged on schedule, with automatic card-on-file update.
- **Delinquency** → 📄 **Dunning notice** → **Collections** (workflow or agency) → **Non-Pay Cancellation** indicator sent to Policy.

## 4. Business rules

### Payment allocation
1. Match the payment to an invoice or instalment using the **policy, account or remittance reference**.
2. Apply it in this priority: **fees & taxes → oldest earned premium → current instalment → future instalment**.
3. Route unmatched or partial-reference payments to **Suspense**.
4. Route residual small balances beyond the **tolerance threshold** to **Write-off** approval.
5. A credit balance triggers an overpayment review, and after approval a **Disbursement** refund.
6. The priority order is **configurable**. CSR and Admin can apply a **manual override**.

### Delinquency state machine
```
Current ──missed instalment──▶ Past Due ──grace days──▶ Delinquent ──unresolved──▶ Non-pay cancel indicator ─▶ Policy
   ▲                                                         │
   └────────────── payment received before cancel effective date (indicator cleared) ─────┘
Delinquent/uncollectable ──approved──▶ Written-off
```
- The grace days are **configurable**.
- Dunning notices go out at **configured delinquency milestones**.
- Every status change is published to Policy.

## 5. External integrations

| Integration | Use |
|---|---|
| Payment gateway / processor | Card authorisation and capture |
| EFT / ACH network | Bank draft submission and settlement file |
| Lockbox / remittance file import | Matching checks and wires |
| Collections agency interface | Referring delinquent accounts |
| GL / Finance posting | Premium, cash and commission entries |
| Fraud / chargeback network | Dispute and reversal notifications |

## 6. Generated documents (through the shared Forms Engine)
Invoice / billing statement packet · Instalment schedule confirmation · Payment receipt/confirmation · Dunning / past-due notice · Commission statement · Refund/disbursement confirmation letter.
Each document is stored, emailed, sent to the print folder and indexed in the DMS. Reissues are versioned; for example, the audit sample shows a corrected commission statement reissued by Admin.

## 7. Personas & dashboards
Each persona gets four widgets: work items, **My tasks**, **Alerts**, and KPIs.

| Persona | Work / queue | Tasks | Alerts | KPIs |
|---|---|---|---|---|
| **Billing CSR** | Open cases (invoice disputes, suspense match requests) | Apply suspense payment, process refund request | NSF/chargeback reversals, overpayment awaiting disbursement | Avg. handle time, same-day resolution % |
| **Underwriter** | Premium exceptions (rate change pending re-bill, OOS premium recalculation) | Approve manual rate-override billing | Endorsement premium variance | Written premium MTD, unbilled premium |
| **Agent** | Commission statements (paid/pending) | Notify customer of instalment due | Client delinquency | Collected premium MTD, commission YTD |
| **Admin** | System queues (suspense unmatched, write-off approvals pending) | Approve write-off batch, reconcile gateway settlement file | Recurring EFT batch failure, allocation rule conflict | Open reconciliation exceptions, auto-matched in last 24h |
| **Customer** | My invoice (amount, due date) | Update card on file | Past-due/dunning notice | Payment history |
| **Management** | Billing KPIs (collected premium MTD, delinquency rate) | Risk: non-pay cancellations (rolling 30 days), write-off amount MTD | Collections escalations | Cycle times: invoice→payment, delinquency→cancellation |

This implies the following approval workflows *(inferred)*: write-off (Admin), disbursement/refund (Admin), manual rate-override billing (Underwriter), and suspense matching (CSR).

## 8. Audit trail
Every billing activity is logged with: **Timestamp · Account/Policy · Activity · Persona · Actor · Result**.
- The actor can be a person or a system component: Billing Batch, Payment Processor, Lockbox Import, Delinquency Batch, Document Batch, Billing Engine, Commission Batch, Card Network.
- Example results: Success, Suspense, Allocated, Past Due, Flagged, Credit, Disbursed, Reversed.
- The audit subject can be an **agent** (e.g. `AGT-2201` for commission), not only a policy.

## 9. Guardrails that apply to Billing
- Billing owns its data. Other domains and the Portal use Billing's APIs and events, never its tables.
- Commands such as take payment, allocate or approve write-off go through **APIs**. Status changes and ledger movements are **published as versioned events**.
- Carry **account, policy, party and correlation IDs** end to end.
- Payments, EFT/ACH files and lockbox imports must be **idempotent**, with retries, dead letters and **reconciliation**.
- Payment data is sensitive PII: tokenise it in the Payment Hub, encrypt it, use least privilege, and audit access.

## 10. Relationship to existing code/docs in this workspace
- `acme.property.us.policy.app/modules/ledger` is a revenue-waterfall/GL module: GWP, policy fee, MGA commission, AIF fee, a rules engine, and a Temporal nightly sweep. See `app/docs/modules/ledger-system/ledger-system-plan.md`. It **overlaps with Billing's commission calc and GL posting**, and it isn't settled which one owns what (see open questions).
- The **policy module already has renewal-billing code**: `logic/billing_queue.go` (`GET /m/policy/billing/queue`) and the page `RenewalBilling.svelte`. Agree with the Policy team who owns these.
- The colleague's PAS new-business plan puts `billing` in `app` and calls it from BindWorkflow (create account → invoice → collect payment → never bind unpaid). **Open decision D-06:** can a policy bind on an unpaid mortgagee escrow invoice? The owner is Billing / Product.
- `app/docs/OVERVIEW.md` already shows a "Billing Service" behind the gateway, with its own schema.
- The renewal programme (`mycel/docs/plan/implementation/`) has billing requirements in **R3 · D3 Billing**:
  - **RNW26**: reissue the renewal bill on relevant change (e.g. mortgagee/payor).
  - **RNW36**: generate renewal billing before expiry on configurable timing.
  - **RNW37**: remove premium-finance data where applicable.
  - Open decisions **D-09** (hold renewal when prior term is in arrears?) and **D-10** (does a reissued bill reset the due date?), and dependency **X-06** (billing plans, payor and fee rules).
  - **RNW74**: Duck Creek stays authoritative for expiring-term billing during coexistence.

## 11. Open questions
1. **Ledger vs Billing ownership:** who owns commission calculation and GL posting — the existing `ledger` module or the new Billing domain?
2. ~~Where does Billing live?~~ **Answered:** as the module `acme.property.us.policy.app/modules/billing`, with its own `billing` schema, according to the colleague's PAS plan. See [conventions.md](../conventions.md) §1. The plan also defines the first billing tables and the BindWorkflow billing steps. See [pas-e2e-implementation-plan.md](../references/pas-e2e-implementation-plan.md) §5.
3. Policy→Billing contract: is it an event, an API, or both? What are the event names and payload schemas, and who owns the schema?
4. Tolerance threshold for write-off, grace days per state/product, and dunning milestones: what are the values, and are they per state or per product?
5. Commission "on earned premium": is it earned pro-rata daily, or based on collected premium? When does chargeback happen on cancellation?
6. Billing screens under Forms, Rating, Integration, User Management, Payment Management and Enterprise Data are still placeholders. Are more specs coming?
7. ~~Is escrow/mortgagee billing and premium finance in scope?~~ **Yes.** Duck Creek has dedicated plans and plan-switching for both. See the rulebook, §2.3.
8. The Duck Creek conflicts are listed in the rulebook, §9: who cancels, the delinquency model, reminders and late charges, commission ownership.
