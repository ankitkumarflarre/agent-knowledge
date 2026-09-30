# Billing Module: End-to-End Implementation Plan

| | |
|---|---|
| Status | **Draft v1** (2026-09-29), for team review |
| Owner | Billing team |
| Builds on | [billing-domain.md](../architecture/billing-domain.md) (target architecture) · [billing-rules-from-duckcreek.md](../architecture/billing-rules-from-duckcreek.md) (business rules) · [conventions.md](../conventions.md) (how we build) · [pas-e2e-implementation-plan.md](../references/pas-e2e-implementation-plan.md) (the Policy team's new-business plan we plug into) |
| Rule references | "R§x" = rulebook section · "A§x" = architecture section · "C§x" = conventions section |

---

## 1. Summary & scope

We are building Billing as a new mycel module, **`acme.property.us.policy.app/modules/billing`**, with its own `billing` schema. The work also adds:
- **Temporal workflows in `wfc`**: account lifecycle, invoice run, payment, follow-up/delinquency, collections and approvals;
- **payment adapters in `integ`**: card gateway, ACH/EFT, lockbox, chargeback and collections agency;
- **BFF routes** and **Billing screens** built from the Acme design system, covering both the internal operations screens **and the Customer/Agent Portal billing screens**. We are part of the Portal team, so we build the portal billing APIs and UI ourselves.

The delivery runs in **8 phases (P0–P7)**. Phase P1 is the thinnest useful slice: **pay-in-full at bind**. It lines up with the Policy team's BindWorkflow phase (their P4), so their new-business journey is never blocked on us. Each later phase adds one capability area: schedules, payments, delinquency, mid-term changes, collections and downstream feeds, then dashboards and hardening.

**In scope:**
- billing accounts and payers (insured, mortgagee/escrow, premium finance)
- pay plans and eligibility
- instalment schedules
- invoicing
- fees and service charges
- all payment methods, recurring EFT and card
- allocation, suspense, overpayment and disbursement
- void and reversal (NSF, chargeback)
- write-off
- delinquency: PCN → NOC / NOL / NOW, and equity cancellation
- automatic reinstatement
- collections
- commission calculation *(ownership pending, decision B-04)*
- Billing↔Policy contracts
- billing documents through the shared forms/document engine
- events to downstream systems
- persona dashboards and the audit trail
- **Portal billing experience, APIs and UI:** customer and agent self-service for viewing invoices, paying, AutoPay, payment methods, history and receipts, pay-plan changes, past-due and cancellation notices with pay-to-cure, and agent commission and client-delinquency views

**Out of scope:**
- **GL posting itself.** Billing emits postings, and the `ledger` module or Finance posts them.
- Building a payment processor. We integrate a vendor gateway.
- **Duck Creek data migration and coexistence cut-over** (decision B-15).
- Claims-account billing.

**Tenancy is a legal requirement, not a filter.** Every billing row carries `tenant_id`, matching `policy.tenant`: today MIE and CURE, with SPIC to follow. Premium cash belongs to the exchange's trust account and the policy fee belongs to the MGA (the Revenue Waterfall). Invoices, allocations and postings must keep them separate.

---

## 2. Where Billing sits

```
Portal (customer/agent, ours) / Ops UI ──► BFF (/v1/…, Idempotency-Key) ──► billing module API (commands, queries)
                                                               │  saves drafts/config only
Policy (BindWorkflow, lifecycle) ──activity calls──► billing   ▼
         ▲                                          wfc: Billing workflows (single writer of statuses)
         │ Policy command API (cancel/reinstate     │  activities ──► integ adapters ──► gateway / ACH / lockbox / agency
         │   requests) + billing.* events           │  activities ──► document module (invoices, notices)
         └──────────────────────────────────────────┘  outbox ──► billing.* events ──► ledger (GL), kpi/ODS, Policy
```

- **Inbound from Policy (A§2):** account setup, premium, fees, pay plan, effective dates, cancellation and reinstatement.
  - For **new business**, Policy's BindWorkflow calls Billing **activities** directly: `CreateBillingAccount`, `CreateInvoice` and `CollectPayment`. This matches the Policy team's plan §6.2.
  - For **mid-term and other lifecycle changes**, Billing consumes Policy **outbox events**: `policy.bound`, `policy.issued`, `policy.endorsed`, `policy.renewed`, `policy.cancelled`, `policy.reinstated`.
- **Outbound to Policy:**
  - `billing.account.status_changed` (Current, Past Due, Delinquent, Written-off, …) as an event.
  - Cancellation and reinstatement go as **commands** to a Policy API: *request non-pay cancellation*, *withdraw cancellation*, *request reinstatement*. Policy stays the only writer of policy transactions. This is decision B-01.

---

## 3. Components & ownership

| Component | Owns | Status |
|---|---|---|
| `app/billing` (module) | Billing schema, config tables, command and query APIs, UI custom element `<billing-app>`, outbox | New |
| `wfc` (billing workflows) | All status transitions and timers (§6) | New workflows. The renewal workflows show the pattern |
| `integ` (billing adapters) | Card gateway, ACH/EFT, lockbox import, chargeback feed, collections agency, print/mail (via document) | New adapters on the shared adapter framework |
| `app/document` (Policy team's) | Rendering, storage and delivery of billing documents | We supply templates and data. The document module is a dependency |
| `app/ledger` (existing) | GL accounts, ledger entries, MGA invoice | We send posting events. Split of work: B-04 |
| `app/policy` (existing) | Policy, term, transactions, `tenant`, and today's `billing_queue` renewal view | Contracts only. Ownership of `billing_queue`: B-12 |
| Portal billing UI | Customer and agent self-service billing screens (§9.2) | New. **We build it.** Where it lives: B-16 |
| `bff` | `/v1/billing/…` and `/v1/portal/billing/…` routes and aggregation, idempotency dedup | Extend. Repo location unknown (B-10) |
| `kpi` | Billing KPI facts from events | Later (P6/P7) |

---

## 4. Data model (`billing` schema)

Conventions (C§3):
- Every table has `tenant_id NOT NULL`.
- Money is `BIGINT` `*_cents`.
- Financial tables are **append-only**; corrections are reversal rows.
- Config is effective-dated, scoped and versioned (DRAFT → APPROVED → PUBLISHED → RETIRED). A term **pins** the config version it started with.

### 4.1 Configuration tables (seeded from the Duck Creek catalog, R§2–R§6)

| Table | Key columns | Seed source |
|---|---|---|
| `account_plan` | code, scope (tenant, state, product, market), description | R§2.1: A1–A10 equivalents |
| `pay_plan` | code, frequency, max_installments, commission_scheme, pcn_allowed, pif_discount_eligible, nb_early_offset_days, rn_early_offset_days, last_due_before_expiry_days, followup_plan_code | R§2.1: P1–P25 equivalents |
| `pay_plan_eligibility` | account_plan_code, pay_plan_code, extra_condition (e.g. `product.advantage = true`) | R§2.2 |
| `down_payment_rule` | pay_plan_code, txn_type, receivable_type, sub_type (wildcard allowed), pct | R§3.3 |
| `plan_switch_rule` | state, product_flag, escrow, mortgagee, premium_finance → target account plan and pay plan | R§2.3 (the endorsement matrix, as a table) |
| `plan_reassign_rule` | pay_plan_group, target_frequency, premium_pct, addon_type (FLAT / MIN_PCT_CAP / BAND), addon params | R§2.4 |
| `fee_rule` | code (INSTALLMENT_SETUP, PREV_BILLED, REINSTATEMENT, SERVICE_CHARGE, …), basis (FLAT / PCT), amount, min, max, scope (state, LOB, form, channel), applies_when | R§4.3 |
| `invoice_threshold` | scope (state, channel), min_amount_cents | R§4.3 |
| `channel_config` | channel (PAPER, EFT, EFT_CARD, EINVOICE overlay), lead_days, retry_count, retry_days, debit_change_threshold_cents | R§4.2 |
| `blackout_calendar` | tenant, date, reason | R§3.2 |
| `allocation_priority` | ordered receivable classes (fees and taxes → oldest earned premium → current → future) | A§4 |
| `tolerance_rule` | write_off_max_cents, dust_write_off_cents, suspense_age_days | R§5 |
| `pcn_plan`, `cancel_notice_plan`, `rescission_plan`, `reinstatement_plan` | trigger days, tolerance $/%, legal and mailing days, max, equity flag; **keyed by state, LOB and product** | R§6.2 (state-keyed, fixing the Duck Creek compliance gap) |
| `collection_plan` | entry days after cancel/expiry, balance start/stop, letter cadence | R§6.6 |
| `recovery_plan` | push_due_date, offsets, redistribution strategy | R§4.4 |
| `aoi_copy_rule` | tenant → interest types that get notice copies | R§6.4 |
| `feature_flag` | reminder_notices, late_charges, … (explicit and visible in admin) | R§6.5 |

### 4.2 Transactional tables

| Table | Key columns | Notes |
|---|---|---|
| `billing_account` | id, tenant_id, account_no, account_type (DIRECT / AGENCY / COLLECTIONS_AGENCY), payer_type (INSURED / MORTGAGEE / PREMIUM_FINANCE), payer_party_id, invoice_method, einvoice, target_due_day, status, delinquency_status | One account can hold several policy terms |
| `billing_term` | id, account_id, policy_id, term_id, account_plan, pay_plan, config_version_id, effective_from/to | The pay-plan history is a new row per change, never an update |
| `bill_item` | id, term_id, txn_type (NB / END / REN / CAN / REIN / AUDIT / MISC / FEE), receivable_type (PREMIUM / POLICY_FEE / TAX / FEE / CHARGE), amount_cents, **revenue_owner** (EXCHANGE_TRUST / MGA / STATE), source_policy_txn_id, effective dates | Receivables. Idempotent on source_policy_txn_id |
| `installment` | id, term_id, seq, due_date, invoice_date, amount_cents by receivable, commission_cents, status (SCHEDULED / INVOICED / PAID / PARTIAL / WAIVED / SUPERSEDED) | A rebuild supersedes rows; it never deletes them |
| `invoice` / `invoice_line` | invoice_no, account_id, invoice_date, due_date, result (SUCCESS / BELOW_THRESHOLD / SUPPRESS_FIRST), out_of_schedule; line_type, amount_cents, revenue_owner, installment_id | The document is generated only on SUCCESS |
| `payment` | id, account_id, method (CARD / ACH / CHECK / WIRE / CASH), amount_cents, gateway_ref, idempotency_key UNIQUE, status (PENDING / CAPTURED / SETTLED / FAILED / REVERSED), received_at | Card and bank data are tokens only |
| `payment_allocation` | payment_id, target (invoice_line / installment / suspense), amount_cents, reverses_allocation_id | Append-only |
| `suspense_item` | payment_id, amount_cents, reason, status, matched_account_id | |
| `disbursement` | account_id, amount_cents, payee_party_id, reason (OVERPAYMENT / CANCEL_RETURN), approval_state, gateway_ref | Needs approval (HITL) |
| `write_off` / `write_off_reversal` | target, amount_cents, reason (DUST / TOLERANCE / UNCOLLECTABLE / CANCEL_FEE), approval_state | Reversed on reinstatement |
| `followup_activity` | account_id, term_id, type (PCN / NOC / NOL / NOW / NOCF / ECN / REIN / RMDR / LTCH / COL_LETTER / COL_TASK), scheduled_for, status (P / I / C / X / F), workflow_id, result | Mirrors the workflow state, for queues and audit |
| `collection_case` | account_id, plan_code, entered_at, balance_cents, step, status, agency_ref | |
| `commission_entry` | term_id, producer_id, basis (EARNED / COLLECTED), amount_cents, statement_id | Depends on B-04 |
| `gateway_event` | payment_id, event_type, raw_ref (integ inbox id) | |
| `billing_outbox` | event_seq, event_type, payload, published_at | Written in the same transaction as the state change |
| `audit_entry` | timestamp, account/policy/agent ref, activity, persona, actor, result, correlation_id | A§8 fields |

Migrations: the first is `0001_init.sql` (core and config) in P0. Later phases add `0002_…` files, additive only, and keep `schema.sql` in step (C§3).

---

## 5. Domain logic (pure functions in `service/logic/`, golden-file tested)

| File | Responsibility | Rules |
|---|---|---|
| `payplan_eligibility.go` | Valid plans for an account plan and product. Enforced on the server | R§2.2 |
| `payplan_switch.go` | Target plan when the billing party changes | R§2.3 |
| `payplan_reassign.go` | Plan change after a partial first payment (threshold formulas) | R§2.4 |
| `schedule_dates.go` | Due and invoice dates: target-day clamp, lead days, early offsets, blackout shift, AGT/COL exemption, precedence | R§3.2 |
| `schedule_amounts.go` | Down payment, even split, remainder on the last instalment, commission split | R§3.3 |
| `fees.go` | Installment setup, previously billed, reinstatement, service charge with collar and exclusions | R§4.3 |
| `invoice_pipeline.go` | Threshold pre-check, suppress-first, service charge, threshold post-check → result | R§4.1 |
| `allocation.go` | Priority order, suspense routing, write-off tolerance, overpayment detection | A§4, R§5 |
| `delinquency.go` | PCN date, tolerance suppression, cancel effective date (greater of equity date and legal + mailing days), NOCF | R§6.2 |
| `reinstatement.go` | Auto-reinstatement eligibility (window, amount covered) | R§7.2 |
| `collections.go` | Entry and exit, cadence step | R§6.6 |
| `revenue_split.go` | Premium → exchange trust, fee → MGA, tax → state, on each invoice line and allocation | §1 tenancy |

None of these functions does I/O. They take config and facts and return decisions. The activities in §6 call them.

---

## 6. Temporal workflows (`wfc/workflows/billing/`)

| Workflow | ID | Lifetime | Signals / updates / queries |
|---|---|---|---|
| **BillingAccountWorkflow** | `billing-account-{accountId}` | The life of the account (continue-as-new each term) | Signals: `bill_items_posted`, `payment_allocated`, `payment_reversed`, `plan_change_requested`, `policy_event`. Query: `getAccountState` |
| **InvoiceRunWorkflow** | `invoice-run-{date}` (Temporal Schedule, nightly) | Minutes | Fans out: installments due for invoicing → `GenerateInvoice` activity per account |
| **PaymentWorkflow** | `payment-{paymentId}` | Minutes to days (ACH settlement) | Signal `gateway_result`, backstop status poll. Saga: capture → allocate → notify. Compensation: reverse the allocation |
| **RecurringDraftWorkflow** | `draft-{installmentId}` | Until settled or retries run out | Retry per `channel_config`, then hands over to delinquency |
| **DelinquencyWorkflow** | `delinquency-{termId}` | From a missed due date until cured or cancelled | Timers: PCN → NOC → cancel request. Signal `payment_received` → NOCF, withdraw (NOW) or cure. Equity-cancellation branch |
| **ReinstatementWorkflow** | `reinstate-{termId}` | Reinstatement window | Auto-reinstate when the amount is covered. Otherwise a task |
| **CollectionsWorkflow** | `collections-{accountId}` | Until the balance is at or below the stop threshold | Letter and task cadence timers. Agency referral |
| **ApprovalWorkflow** (generic) | `approval-{kind}-{id}` | Until a human decides | For write-offs, disbursements and rate-override billing. Signal `billing_decision_recorded`, four-eyes where configured |
| **ReconciliationWorkflow** | `recon-{gateway}-{date}` (scheduled) | Minutes | Import the settlement file, auto-match, raise exceptions |

Workflow rules (C§5):
- Everything is deterministic. All I/O happens in activities.
- Each workflow is the only writer of the statuses it owns.
- Every step has a compensation, and a compensation is a reversal.
- Search attributes `MycelState` and `MycelPhase` are set at each phase.
- Everything is registered in `register.go` and runs on the task queue `acme-property-us-policy-tasks`.
- **No silent failure:** bounded retry, then an alert, then a human task (R§1.4).

---

## 7. Integrations (`integ` adapters)

| Adapter | Used for | Degradation if the vendor is down | Phase |
|---|---|---|---|
| Card gateway (hosted fields, tokens) | NB pay-in-full, recurring card, refunds | Retry, then refer. **Never mark paid without confirmation** | P1 |
| ACH/EFT processor | Bank drafts, settlement and return files (NSF) | Retry the batch, then alert Admin | P2 / P3 |
| Lockbox / remittance import | Checks and wires | File-level retry. Unmatched items go to suspense | P3 |
| Chargeback / dispute feed | Reversals | Webhook inbox with dedup and a status poll | P3 |
| Collections agency | Referring delinquent accounts | Queue and retry | P6 |
| Print/mail, e-delivery (via the document module) | Invoices, notices, letters | Retry, then fallback channel. **The mailed date is the evidentiary timestamp** | P1 onward |

All adapters use one contract (C§6): a normalised result with status OK, PARTIAL or UNAVAILABLE, and a `vendor_call` log entry tagged with the correlation ID. Webhooks go through `integ.webhook_inbox`, then to the billing module, then to a workflow signal. Vendor choice is decision B-09.

---

## 8. APIs

### 8.1 BFF routes (`/v1`, every write requires an `Idempotency-Key`)

| Route | Kind | Purpose |
|---|---|---|
| `GET /v1/billing/pay-plans?policyContext…` | Read | Eligible plans for the quote wizard's payment tab |
| `POST /v1/quotes/{id}/payments` | Write | Hosted-payment session. Already in the Policy team's plan; we implement it |
| `GET /v1/billing/accounts/{id}` | Aggregation | Account workspace: terms, schedule, invoices, payments, activities |
| `GET /v1/billing/accounts/{id}/schedule/preview` | Read | Projected schedule: a what-if for a proposed change |
| `POST /v1/billing/accounts/{id}/payments` | Write | Take a payment (CSR; the portal uses §8.1a) |
| `POST /v1/billing/accounts/{id}/payment-method` | Write | Update card or bank on file (token) |
| `POST /v1/billing/accounts/{id}/plan-change` | Write | Request a pay-plan change. Validated on the server |
| `GET /v1/billing/suspense` · `POST /v1/billing/suspense/{id}/match` | Queue / write | Suspense worklist |
| `GET /v1/billing/approvals` · `POST /v1/billing/approvals/{id}/decision` | Queue / write | Write-off, disbursement and rate-override approvals |
| `GET /v1/billing/delinquency` · `GET /v1/billing/collections` | Queue | Worklists |
| `GET /v1/billing/dashboard?persona=` | Aggregation | Persona widgets (A§7) |
| `GET/POST /v1/admin/billing/config/{table}/versions` (+ `/approve`, `/publish`) | Admin | Config versions, four-eyes |
| `GET /v1/policies/{id}/360` | Aggregation | The Policy team's route; we supply its Billing tab |

### 8.1a Portal routes (`/v1/portal/billing/…`, customer and agent self-service)

The caller's identity scopes every portal route on the **server**:
- a **customer** sees only the accounts where they are the payer or insured;
- an **agent** sees only accounts in their book;
- the customer view never includes internal fields such as internal notes, thresholds or allocation detail (C§4 "two views").

| Route | Kind | Purpose |
|---|---|---|
| `GET /v1/portal/billing/summary` | Aggregation | Home widget: amount due, next due date, status, AutoPay on/off, alerts |
| `GET /v1/portal/billing/accounts/{id}/invoices` · `…/invoices/{invoiceId}/document` | Read | Invoices and statements, with a PDF from the document module |
| `GET /v1/portal/billing/accounts/{id}/schedule` | Read | Upcoming instalments |
| `POST /v1/portal/billing/accounts/{id}/payment-sessions` | Write | One-time payment through the gateway's hosted fields (card or ACH) |
| `GET /v1/portal/billing/accounts/{id}/payments` · `…/payments/{pid}/receipt` | Read | Payment history and receipts |
| `PUT /v1/portal/billing/accounts/{id}/autopay` | Write | Enrol in or cancel AutoPay (recurring EFT or card), with an e-sign authorisation record |
| `PUT /v1/portal/billing/accounts/{id}/payment-method` | Write | Replace the card or bank on file (token only) |
| `GET /v1/portal/billing/accounts/{id}/pay-plans` · `POST …/plan-change-requests` | Read / write | Eligible plans and a request to change plan (server-side validation, with a fee preview) |
| `GET /v1/portal/billing/accounts/{id}/notices` | Read | Past-due, PCN, cancellation and withdrawal notices, with the pay-to-cure amount and deadline |
| `GET /v1/portal/billing/agent/clients?status=` | Read (agent) | Book billing: client delinquency and instalments due |
| `GET /v1/portal/billing/agent/commission-statements` | Read (agent) | Commission statements, depending on B-04 |
| `POST /v1/portal/billing/agent/clients/{accountId}/payments` | Write (agent) | Agent takes a payment for a client |

### 8.2 Events we publish (versioned, via outbox)

`billing.account.created` · `billing.invoice.issued|suppressed` · `billing.payment.received|allocated|reversed` · `billing.account.status_changed` · `billing.pcn.issued` · `billing.cancellation.requested|withdrawn` · `billing.reinstatement.requested` · `billing.writeoff.posted` · `billing.disbursement.paid` · `billing.commission.calculated` · `billing.posting.created` (to ledger/GL)

### 8.3 Commands we need from Policy (B-01)

`RequestNonPayCancellation(policyId, termId, cancelEffectiveDate, reason, noticeRef)` · `WithdrawCancellation(…)` · `RequestReinstatement(…)` · `RecordPCN(…)`. Policy starts its own transaction and replies with an event.

---

## 9. Screens

### 9.1 Internal (operations) screens

All screens are Svelte 5 inside `<billing-app>`, built from the Acme composite patterns (C§7). Billing needs its own accent colour (B-14).

| # | Screen | Pattern | Personas | Phase |
|---|---|---|---|---|
| 1 | Payment tab (quote wizard, tab 6) | Form + hosted fields | Agent, Customer | P1 (with Policy) |
| 2 | Billing tab in Policy 360 | Detail tab | All internal | P1 |
| 3 | Billing Account Workspace (terms, schedule, invoices, payments, activities, audit) | Detail/Case + 320px rail | CSR, Admin, Underwriter | P2 |
| 4 | Invoice & Schedule view, with projected-schedule preview | Financial/Estimate | CSR, Agent | P2 |
| 5 | Take Payment / Update Payment Method | Modal | CSR | P3 |
| 6 | Suspense Queue + Match | Queue/Worklist + drawer | CSR, Admin | P3 |
| 7 | Approvals Queue: write-off, refund, rate override | Queue + HITL decision modal | Admin, Underwriter | P3 |
| 8 | Reconciliation Exceptions | Queue/Worklist | Admin | P3 |
| 9 | Delinquency & Notices Monitor (PCN / NOC / NOL, mailed dates) | Queue/Worklist | CSR, Admin, Agent | P4 |
| 10 | Collections Worklist | Queue/Worklist | CSR, Admin | P6 |
| 11 | Commission Statements | Queue + detail | Agent, Admin | P6 |
| 12 | Persona Dashboards (CSR, UW, Admin, Management; customer and agent dashboards are portal screen P-12) | Dashboard | All | P7 |
| 13 | Billing Config Manager (plans, fees, notice plans: versions, simulate, four-eyes) | Queue + Detail | Admin | P7 (seed migrations until then) |

### 9.2 Portal (customer and agent self-service) screens

We build these. They call only the `/v1/portal/billing/…` routes (§8.1a). They follow the Acme design tokens, adapted for a customer-facing audience (B-16). Accessibility (WCAG 2.1 AA) and mobile layouts are required.

| # | Screen | Audience | Phase |
|---|---|---|---|
| P-1 | Pay at bind: the quote wizard payment tab, full pay (shared with #1) | Customer, Agent | P1 |
| P-2 | Billing Summary: amount due, next due date, status, alerts | Customer | P1 |
| P-3 | Pay Now: one-time card or ACH payment, then a receipt | Customer | P1 |
| P-4 | Invoices & Statements (PDF download) and Upcoming Schedule | Customer | P2 |
| P-5 | AutoPay enrolment and Payment Method management | Customer | P2 |
| P-6 | Payment History & Receipts | Customer | P3 |
| P-7 | Pay-Plan Change request (eligible plans, fee preview) | Customer, Agent | P3 |
| P-8 | Past-Due & Cancellation Notices, with a pay-to-cure call to action and a reinstatement payment | Customer | P4 |
| P-9 | Agent Book Billing: client delinquency, instalments due, take payment for a client | Agent | P4 |
| P-10 | Mortgagee / premium-finance payer view (read-only invoice for the escrow payer) | Customer | P5 |
| P-11 | Agent Commission Statements | Agent | P6 |
| P-12 | Customer and Agent billing dashboards (the A§7 customer and agent widgets) | Customer, Agent | P7 |

---

## 10. Phased delivery

Each phase ends with an **exit gate**: a scenario that passes end to end on the cluster (`mycel up --build`). Merged code alone doesn't count. The team estimates durations.

### P0: Foundations
- **Backend:**
  - `mycel add module billing` and the module layout (C§1).
  - `0001_init.sql`: account, term, bill_item, invoice, payment, allocation, outbox and audit, plus all config tables in §4.1 with version and pinning.
  - The money type.
  - The error envelope.
  - Tenant FK.
  - Config publish API (seed via migrations).
- **Workflows and integrations:** the billing workflow package skeleton in `wfc`, and a card-gateway adapter stub on the `integ` framework.
- **Screens:** the module shell with its accent colour. The portal billing shell, with authentication and routing wired to customer and agent identities (B-16).
- **Exit gate:** the module runs on the cluster, the seed catalog loads, and a golden test of `schedule_amounts` passes to the cent.
- **Blocked by:** B-04 (schema boundary with ledger), B-11 (tenant codes), B-16 (where the portal UI lives).

### P1: Pay-in-full at bind (lines up with the Policy team's P4)
- **Backend:**
  - The `CreateBillingAccount`, `CreateInvoice` and `CollectPayment` activities, called by BindWorkflow.
  - Invoice lines with the **revenue split**.
  - Idempotent payment recording.
  - `billing.*` events.
  - Pay-plan eligibility endpoint, full-pay only.
- **Workflows and integrations:**
  - PaymentWorkflow (card).
  - Gateway adapter with hosted fields, webhook and status poll.
  - Compensations `VoidInvoice` and `RefundPayment`.
- **Screens:** Payment tab (with the Policy team) and the Billing tab in Policy 360. **Portal:** P-1 Pay at bind, P-2 Billing Summary, P-3 Pay Now.
- **Exit gate:** Maria's HO3 binds after a card payment, and she can then see the amount due and pay again from the portal. The invoice is PAID and split between exchange trust, MGA and state. A duplicate webhook or a retried activity never double-charges. A gateway timeout never binds unpaid.
- **Blocked by:** B-08 (escrow bind, D-06), B-09 (gateway vendor), B-10 (bff).

### P2: Instalment plans & invoicing engine
- **Backend:**
  - The full pay-plan catalog.
  - Server-side eligibility.
  - `schedule_dates` and `schedule_amounts`.
  - Installment setup fee.
  - `invoice_pipeline` (threshold, suppress-first, service charge).
  - Automatic plan reassignment after the first payment.
  - Projected-schedule preview.
- **Workflows and integrations:**
  - BillingAccountWorkflow.
  - InvoiceRunWorkflow (nightly schedule).
  - RecurringDraftWorkflow for card and EFT, with the ACH adapter.
  - Invoice document through the document module.
- **Screens:** Account Workspace and Invoice & Schedule view. **Portal:** P-4 Invoices & Schedule, P-5 AutoPay & Payment Method.
- **Exit gate:**
  - Semi-annual and quarterly plans bind with the correct down payment and fee.
  - Invoices generate on their invoice dates, and invoices below the threshold are suppressed while their follow-ups are still scheduled.
  - A partial first payment re-plans correctly at the threshold boundary.
  - All of this is golden-tested per state.
- **Blocked by:** B-06 (SC/NC instalments).

### P3: Payments, allocation & exceptions
- **Backend:**
  - All payment methods.
  - `allocation.go`.
  - Suspense.
  - Overpayment detection and disbursement.
  - Void and reversal (NSF, chargeback) with allocation reversal.
  - Write-off (dust and tolerance).
  - Debit change notice.
- **Workflows and integrations:**
  - ApprovalWorkflow for write-offs and disbursements.
  - ReconciliationWorkflow.
  - Lockbox import and chargeback feed.
  - ACH return file.
- **Screens:** Take Payment, Suspense Queue, Approvals Queue and Reconciliation Exceptions. **Portal:** P-6 Payment History, P-7 Pay-Plan Change.
- **Exit gate:**
  - A payment with an unknown reference lands in suspense, is matched by a CSR and is allocated in priority order.
  - An NSF after allocation reverses it and re-opens the instalment.
  - An overpayment is refunded only after Admin approval.
  - The settlement file auto-matches.
- **Blocked by:** B-09 (ACH and lockbox vendors).

### P4: Delinquency, notices & reinstatement
- **Backend:**
  - `delinquency.go` with **state-keyed** notice plans.
  - PCN tolerance.
  - Cancellation date = greater of the equity date and the legal + mailing date.
  - NOCF, NOW and equity cancellation.
  - `reinstatement.go`.
  - `billing.account.status_changed` events.
  - Use of the Policy command API.
- **Workflows and integrations:**
  - DelinquencyWorkflow.
  - ReinstatementWorkflow.
  - Notice documents with AOI copies.
  - Mailed-date evidence.
- **Screens:** Delinquency & Notices Monitor. **Portal:** P-8 Notices & pay-to-cure, P-9 Agent Book Billing.
- **Exit gate:**
  - A missed instalment leads to a PCN on the configured day, then a NOC after the legal and mailing days, then a cancellation request accepted by Policy.
  - A payment before the effective date sends a NOW and withdraws the cancellation.
  - A full payment within the window reinstates automatically; a partial payment raises a task.
- **Blocked by:** B-01 (who cancels), B-02 (status model), B-03 (reminders and late charges), B-07 (legal notice periods).

### P5: Mid-term lifecycle, renewal, mortgagee & premium finance
- **Backend:**
  - Item-level schedule rebuild for endorsements, audits and miscellaneous items.
  - `payplan_switch.go` for mortgagee/escrow and premium-finance changes.
  - Previously-billed fee.
  - Cancellation effects (write off the setup fee or suspend, hold overpayment, dust write-off).
  - Reinstatement reverses write-offs.
  - Renewal billing on a configurable lead time.
  - Out-of-sequence re-sequencing.
  - Mortgagee invoice.
- **Workflows and integrations:** policy-event consumers, the renewal invoice schedule, and delivery to LexisNexis for mortgagees.
- **Screens:** Account Workspace additions: payer changes and history. **Portal:** P-10 Mortgagee/premium-finance payer view.
- **Exit gate:**
  - Adding an escrow mortgagee switches the plan and bills the mortgagee; removing it reverts.
  - A backdated endorsement after renewal re-sequences correctly.
  - The renewal bill issues on its lead time and is reissued when the payor changes (RNW26/36/37).
- **Blocked by:** B-05 (renewal forced to Annual), B-12 (`billing_queue` ownership).

### P6: Collections, commission & downstream
- **Backend:**
  - `collections.go` (entry, exit, cadence, duplicate guard, transfer suppression).
  - Commission calculation and statements *(or events to ledger, per B-04)*.
  - `billing.posting.created` feed to the ledger.
  - CDC/events to kpi and ODS.
- **Workflows and integrations:** CollectionsWorkflow and the collections-agency adapter.
- **Screens:** Collections Worklist and Commission Statements. **Portal:** P-11 Agent Commission Statements.
- **Exit gate:**
  - A cancelled term with a balance above the threshold enters collections, and its letters go out on cadence.
  - Agency referral works.
  - Commission statements tie out to earned premium.
  - Ledger postings reconcile to the billing subledger.
- **Blocked by:** B-04 (commission and GL ownership), B-13 (collections stop threshold).

### P7: Dashboards, config manager & hardening
- **Backend:** persona dashboard aggregations and the config simulation (replay against recent accounts).
- **Workflows and integrations:** load test of the nightly invoice run, dead-letter-queue alerting, runbooks.
- **Screens:** Persona Dashboards and the Billing Config Manager. **Portal:** P-12 Customer and Agent dashboards; accessibility and performance hardening.
- **Exit gate:** every persona in A§7 sees its widgets from live data. A config change goes through publish without a code deploy. The nightly run meets its SLA at the target volume.
- **Blocked by:** B-15 (Duck Creek coexistence) before go-live.

---

## 11. Test strategy

- **Golden files (to the cent), per tenant × state × plan** (R§1.7):
  - down payment
  - instalment split and remainder
  - fees
  - thresholds
  - reassignment boundaries (≥ vs >)
  - allocation order
  - cancellation date
  - revenue split
- **Date tests:** 31st due day in February, blackout on the due date, AGT/COL accounts skipping lead days, missed-installment catch-up, short terms.
- **Failure tests:**
  - duplicate webhook
  - activity retried after capture
  - gateway, ACH or document outage
  - NSF after allocation
  - chargeback after write-off
  - Policy command API unavailable during cancellation (it must retry and alert, never cancel silently or skip)
- **Workflow tests** (Temporal test suite): timers for PCN → NOC → cancel; a payment signal racing the cancellation timer; continue-as-new at renewal.
- **Portal security tests:** customer A can never read or pay customer B's account; an agent sees only their book; customer responses never contain internal fields; card and bank data only ever pass through hosted fields; the portal meets WCAG 2.1 AA and works on mobile.
- **Regression guards:** a disabled feature (reminders, late charges) never fires. An invalid plan is rejected by the API, not just hidden in the UI.
- **Reconciliation:** the billing subledger equals the ledger postings each day.

---

## 12. Non-negotiables & edge cases

1. **Never bind unpaid. Never mark paid without gateway confirmation.** Payment ≠ issuance.
2. **Tenant segregation:** exchange premium, MGA fee and state tax never mix in one allocation or posting.
3. **No silent failure:** every cross-domain call has a retry, a dead-letter queue, an alert and a human task.
4. **Money is exact:** integer cents, deterministic rounding, append-only ledger-style records.
5. **Config, not code:** no state, tenant or plan `if`s. Notice periods are state-keyed with legal sign-off.
6. **Portal access is scoped on the server:** a customer sees only their own accounts and an agent only their book. The UI never does the filtering.
7. **Explainable:** every invoice, allocation, notice and status change traces to its config version, inputs, actor and correlation ID.

**Edge cases to handle:**
- A payment arrives the day the cancellation takes effect.
- A partial payment after a PCN.
- Two payments race one instalment.
- A reinstatement is paid from suspense.
- A backdated endorsement after an invoice was issued.
- The payer switches from insured to mortgagee mid-term.
- A refund is due on a cancelled term that has an open chargeback.
- The nightly run crashes partway through (it must resume, not re-run).
- A tenant's trust account can't be resolved (block the run and alert).

---

## 13. Open decisions

| ID | Decision | Blocks | Owner |
|---|---|---|---|
| B-01 | Who starts a non-pay cancellation or reinstatement: a Billing command to a Policy API (proposed) or Billing driving the Policy transaction (Duck Creek)? | P4 | Billing + Policy |
| B-02 | One delinquency status model: architecture statuses ↔ PCN / NOC / NOL / equity | P4 | Billing + Policy |
| B-03 | Reminder notices and late charges: in scope? (Disabled in Duck Creek.) | P4 | Business |
| B-04 | Commission and GL posting: Billing vs. the existing `ledger` module. Proposed: Billing = subledger and commission calculation, Ledger = GL | P0 schema, P6 | Billing + Ledger owner |
| B-05 | Renewal forced to Annual, or the insured's plan carries over? | P5 | Business |
| B-06 | SC/NC instalment plans: by design or unfinished? | P2 | Business / Compliance |
| B-07 | Legal notice, mailing and cure days per state/LOB | P4 | Compliance |
| B-08 | = Policy plan D-06: bind on an unpaid mortgagee escrow invoice? | P1 | Billing / Product |
| B-09 | Vendors: card gateway, ACH/EFT, lockbox, collections agency | P1 / P3 / P6 | Integration / Procurement |
| B-10 | Where the `bff` repo lives, and who owns billing routes there | P1 | Platform |
| B-11 | Tenant codes: Duck Creek uses Cajun = "CIC", the policy module uses "CURE". SPIC isn't seeded yet | P0 | Policy + Billing |
| B-12 | Move `policy/billing_queue.go` and `RenewalBilling.svelte` into billing? | P5 | Billing + Policy |
| B-13 | Collections stop threshold (Duck Creek's $99,999 means never) | P6 | Business |
| B-14 | Billing domain accent colour | P0 | UX |
| B-15 | Duck Creek coexistence: which system bills which terms, and cut-over/migration (RNW74) | Before go-live | Leadership / IT |
| B-16 | Portal UI: where it lives (a portal app or module vs. a view of `<billing-app>`), the customer-facing style on top of the Acme tokens (the design system targets dense ops screens), and customer and agent login through `iam` | P0 / P1 | Portal team (us) + Platform + UX |

---

## 14. Risks

| Risk | Mitigation |
|---|---|
| The Policy team's BindWorkflow timeline drives our P1 | Agree the activity contract in P0. Stub Billing activities so they aren't blocked |
| The document module doesn't exist yet (Policy plan P5) | Render a minimal invoice PDF behind the same interface. Switch over when the module lands |
| Notice periods aren't legally validated per state | P4 is gated on B-07. Seed data carries a `legal_approved` flag |
| Duplicated commission logic with `ledger` | Settle B-04 before P0 migrations |
| The nightly invoice run doesn't scale | Fan out per account in Temporal, rate-limit per adapter, load test in P7 |
| Scope creep toward a Duck Creek replica | Rulebook §8 anti-pattern checklist in code review |
