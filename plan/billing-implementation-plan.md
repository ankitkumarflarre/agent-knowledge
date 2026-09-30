# Billing Module: End-to-End Implementation Plan

| | |
|---|---|
| Status | **Draft v2** (2026-09-30): **re-based on the Project Siddhi Phase 1 backlog**. v1 (2026-09-29) assumed new business and is superseded. See the change log in §15 |
| Owner | Billing team (also part of the Portal team: we build the billing APIs and UI, including the Customer/Agent Portal billing screens) |
| Governing scope | [project-siddhi-phase1-backlog.md](../references/project-siddhi-phase1-backlog.md): **Stages 5–7, features 34–62, plus feature 70** |
| Builds on | [billing-domain.md](../architecture/billing-domain.md) (target architecture) · [billing-rules-from-duckcreek.md](../architecture/billing-rules-from-duckcreek.md) (rules baseline) · [conventions.md](../conventions.md) (how we build) |
| References | "F#" = Siddhi feature number · "R§x" = rulebook section · "A§x" = architecture section · "C§x" = conventions section |

---

## 1. Summary & scope

**What Phase 1 is:**
- **Manatee, Florida, Personal Home & Dwelling.**
- **Renewal processing only**: policies move from Duck Creek into Siddhi at their renewal.
- **First renewal offer: April 2027.** The first renewed terms take effect around July 2027.
- Duck Creek and Siddhi run side by side for 12–16+ months, and Duck Creek stays authoritative for each expiring term.

Billing's job is Stages 5–7 of the lifecycle:
- **Billing Establishment:** set up the billing relationship when a renewal is **committed** (F24).
- **Invoicing:** tell the right payer what they owe.
- **Payment Processing:** receive, apply, reconcile, refund, pay commission, and report.

It also covers **flat cancellation when a renewal invoice is unpaid**, plus reinstatement (F70).

**What we build:**
- The module **`acme.property.us.policy.app/modules/billing`**, with its own `billing` schema.
- **Temporal workflows in `wfc`**, hooked into the existing Policy `RenewalWorkflow` at commit.
- **`integ` adapters** for Heartland, the RT Lawrence lockbox, FileSphere, Workday and the commission ACH file.
- **BFF routes.**
- **Internal billing screens and the Portal billing screens.**

**Rules baseline:** the current **Duck Creek billing configuration** is the default. The rules are extracted, checked with the client and loaded as versioned config, not redesigned (Siddhi scope decision; rulebook R§1–R§7). For Phase 1 only the **FL Manatee** plans are needed:
- Annual (full pay)
- Semi-annual (2-pay, 60% down)
- Quarterly (4-pay, 40% down)
- Mortgagee/escrow full-pay
- Premium-finance full-pay

**In scope (Day 1 unless marked):**

| Stage | Capabilities |
|---|---|
| **5 Establishment** | Choose the payer independently of who's on the policy (F34). **Keep the insured's pay plan at renewal** and auto-assign 2-pay/4-pay (F36). Mortgagee change vs. billing-only "stop billing my mortgage" (F37). **Equity billing and prior-term balance** (F38). **Commission dated by the policy effective date** (F40). **FL statutory fees and assessments** (FHCF, FIGA, policy fee) and how they're spread over instalments (F45). **Data foundation for reporting** (F44). *Confirm:* premium finance config (F35), agency transfer commission (F39), parent-child commission (F41), special rates (F42), MGA → Company → Agency hierarchy (F43) |
| **6 Invoicing** | **Invoice shows the actual plan plus the other available plans** (F46). **Recalculate after an endorsement, keeping the original** (F47). *Confirm:* Heartland schedule sync (F48), recurring-payment notices (F49) |
| **7 Payments** | **Heartland card, RT Lawrence lockbox, ACH and premium finance** (F50). **Split lockbox files between Duck Creek and Siddhi** (F51). **Research bucket, multi-policy and cross-company checks, house account** (F52). **Reverse, transfer or refund** (F53). **Daily cash-and-suspense report; accounting authorises before a refund is released** (F54). **Spread endorsement premium over the remaining instalments** (F55). **Commission calculation, ACH file and clearance** (F58). **Commission statements with a real CSV/Excel export** (F59). **Receivables and commission reconciliation** (F61). *Confirm:* Invoice Cloud refunds (F57), collections (F60), Workday GL, ISO and FHCF reporting (F62) |
| **8 (Billing side)** | **Flat cancellation for non-payment of the renewal invoice**, carried-forward PCN, lapse and reinstatement (F70). Shared policy and billing notes (F72). Agent vs. accounting billing views (F67). Accounting/treasury role (F74) |
| **Portal (ours)** | Billing summary, invoices and plan options, pay now (Heartland hosted fields), history and receipts, notices with pay-to-cure, agent book billing, agent commission statements |

**Out of scope for Phase 1:**
- **New-business billing** (no NB quoting in Phase 1).
- **Recurring-payment collection and AutoPay enrolment** (F56: Safepoint runs this directly with Heartland; we only feed the instalment amount and due date).
- **Queue and task management** (FileSphere owns it; we only trigger tasks and read back their status).
- The data warehouse (Safepoint's).
- The Workday GL itself.
- Claims billing.
- Non-FL and non-Manatee rules. The model stays multi-tenant and multi-state, but we only seed FL Manatee.

**Tenancy:** every billing row carries `tenant_id`. Phase 1 = **MIE** (Manatee). Premium, the MGA policy fee and state assessments (FHCF, FIGA) belong to different owners, and invoices, allocations and postings keep them apart.

---

## 2. Where Billing sits

```
Duck Creek (expiring term, authoritative) ──► Conversion (Policy team, F6–F13) ──► Siddhi renewal term
                                                                                  │
Policy RenewalWorkflow ── F24 renewal commit ──► EstablishRenewalBilling (billing activity)
                                                                                  ▼
Portal (ours) / Ops UI ──► BFF (/v1…, Idempotency-Key) ──► billing module ◄── wfc billing workflows (single writer)
                                                          │                       │
                    Heartland (card, refunds, schedule feed) ◄── integ ──► RT Lawrence lockbox files (split DC/Siddhi)
                    FileSphere (documents + task triggers)   ◄── integ ──► Workday GL · commission ACH / clearance files
                                                          │
                                  outbox ──► billing.* events ──► Policy (status, non-pay) · reporting replica (F44)
```

- **Inbound from Policy:**
  - At **renewal commit** (F24), the Policy `RenewalWorkflow` (`wfc/workflows/renewal`) calls the Billing activity `EstablishRenewalBilling`. The term snapshot it passes carries the premium, fees and assessments, effective dates, parties (insured, mortgagee, premium finance), the **current pay plan and payer carried over from Duck Creek**, and the producer/agency.
  - Mid-term changes (endorsements, mortgagee change, cancellation, reinstatement) reach Billing as Policy **outbox events**.
- **Outbound to Policy:**
  - `billing.account.status_changed` events.
  - Cancellation and reinstatement go as **commands** to a Policy API: request a flat non-pay cancellation, withdraw it, or request reinstatement. Policy stays the only writer of policy transactions (B-01).
- **Duck Creek coexistence:**
  - Billing reads the cross-system routing list (F10, owned by the Conversion team) to know which system owns a policy and term. It needs that for lockbox splitting (F51) and for prior-term balances (F38).
  - Siddhi never posts to a Duck Creek term. Anything touching the prior term goes back to Duck Creek (B-20).

---

## 3. Components & ownership

| Component | Owns | Status |
|---|---|---|
| `app/billing` (module) | Billing schema, config, command and query APIs, internal UI `<billing-app>`, outbox, reporting extract | New |
| `wfc` billing workflows | All billing status transitions and timers (§6) | New. Hooks into the existing `renewal` workflow |
| `integ` billing adapters | Heartland, RT Lawrence lockbox, FileSphere (tasks and documents), Workday, commission ACH/clearance files, Invoice Cloud (confirm), collections agency (confirm) | New, on the shared adapter framework |
| FileSphere (Safepoint platform) | **Document system of record, and all queues and tasks** | External. We trigger, store references and consume statuses |
| Heartland (Safepoint vendor) | Card capture and refunds; **recurring collection is Safepoint-run** | External. We capture one-time payments and feed schedules |
| `app/policy` + Policy `RenewalWorkflow` | Terms, transactions, commit (F24), cancellation and reinstatement transactions, `tenant`, today's `billing_queue` | Contracts. `billing_queue` ownership: B-12 |
| `app/ledger` (existing) | Revenue waterfall and ledger entries | **Role in Phase 1 unclear now that the GL is Workday.** B-04 / B-22 |
| Portal billing UI | Customer and agent billing screens (§9.2) | New. **We build it.** Where it lives: B-16 |
| `bff` | `/v1/billing/…` and `/v1/portal/billing/…` routes, idempotency dedup | Extend. Repo location: B-10 |

---

## 4. Data model (`billing` schema)

Conventions (C§3):
- Every table has `tenant_id NOT NULL`.
- Money is `BIGINT` `*_cents`.
- Financial tables are append-only; corrections are reversals.
- Config is effective-dated, versioned and pinned per term.
- **Business time comes from the request's temporal context** (the Siddhi time-travel framework, C§5), never from the server clock.

### 4.1 Configuration (seeded from extracted Duck Creek config, validated by the client)

| Table | Key columns | Phase 1 seed |
|---|---|---|
| `account_plan`, `pay_plan`, `pay_plan_eligibility` | frequency, max instalments, down payment %, commission scheme, PIF discount flag, offsets | FL Manatee: Annual, 2-pay 60%, 4-pay 40%, mortgagee full-pay, premium-finance full-pay (R§2.1) |
| `down_payment_rule`, `plan_reassign_rule` | Down payment by transaction and receivable type. Thresholds for auto 2-pay/4-pay (F36, R§2.4) | FL formulas |
| `plan_switch_rule` | (escrow, mortgagee, premium finance) → target plan (R§2.3) | FL rows |
| `fee_rule` | code, basis FLAT/PCT, amount, **payee/revenue_owner (CARRIER / MGA / FHCF / FIGA / STATE)**, **instalment distribution rule**, scope, effective dates | **Policy fee, FHCF emergency assessment, FIGA assessment** (F45), installment setup fee, others extracted from Duck Creek |
| `invoice_threshold`, `channel_config`, `blackout_calendar` | R§3–R§4 | FL values |
| `allocation_priority`, `tolerance_rule` | Allocation order. Dust and write-off tolerances. **High-value refund authorisation threshold (F54)** | Extracted plus client-set |
| `endorsement_spread_rule` | Default: spread over remaining instalments. Override distributions such as 40/20/20/20 (F55) | Client-set |
| `equity_rule` | Modified equity earnings method parameters. Equity-dating deadline (about 20 days) (F38) | **To be specified (B-19)** |
| `cancel_notice_plan`, `pcn_plan`, `reinstatement_plan` | State/LOB-keyed notice, grace and cure days. **Renewal non-pay leads to FLAT cancellation** (F70) | FL rules extracted from Duck Creek (B-07) |
| `commission_plan`, `commission_rate` | Rate by **policy effective date** and plan effective date (F40). Agency and risk-origin exceptions (F42) | From Duck Creek and accounting |
| `premium_finance_config` | Enabled by state/MGA, refund and routing rules (F35) | Confirm |
| `feature_flag` | reminder notices, late charges, collections, Invoice Cloud, … | Explicit, visible to admins |

### 4.2 Transactional tables

| Table | Key columns | Notes |
|---|---|---|
| `billing_account` | tenant_id, account_no, account_type, status, delinquency_status | |
| `bill_to_assignment` | term_id, payer_type (INSURED / MORTGAGEE / PREMIUM_FINANCE), payer_party_id, effective_from, source (POLICY_ENDORSEMENT / BILLING_ONLY) | **Independent of who's on the policy** (F34). Billing-only change vs. mortgagee change from Policy (F37). A new row per change |
| `billing_term` | account_id, policy_id, term_id, **source_system (DC / SIDDHI)**, pay_plan, config_version_id, producer/agency_id, effective dates | The pay plan carries over from Duck Creek at renewal (F36) |
| `bill_item` | txn_type, receivable_type (PREMIUM / POLICY_FEE / FHCF / FIGA / TAX / FEE), amount_cents, revenue_owner, source_policy_txn_id | Idempotent on the source transaction |
| `installment` | seq, due_date, invoice_date, amounts by receivable, status (… / SUPERSEDED) | Endorsement spread (F55) creates superseding rows |
| `invoice`, `invoice_line` | result, **plan_options_snapshot** (F46), **supersedes_invoice_id** (F47), delivery refs (FileSphere doc id, channel, sent/mailed timestamps) | The original is kept on recalculation |
| `heartland_schedule_sync` | term_id, amount_cents, due_date, sent_at, ack_status | Outbound feed log (F48) |
| `payment` | channel (HEARTLAND_CARD / LOCKBOX_CHECK / ACH / PREMIUM_FINANCE / WIRE), amount_cents, source_file_id, external_ref, idempotency_key, status | |
| `lockbox_file`, `lockbox_item` | file id, item, **routing (DC / SIDDHI / HOUSE / RESEARCH)**, remitter, references | Duck Creek/Siddhi split with reconciliation (F51) |
| `payment_allocation` | target, amount_cents, reverses_allocation_id, **transfer_id** | Append-only. A direct policy-to-policy transfer is a linked pair, not a reversal (F53) |
| `research_item` (suspense) | payment_id, reason, status, FileSphere task ref | Research bucket (F52) |
| `house_account_entry` | payment_id, other_insurer, amount_cents, return_check_ref | Mortgage checks that cover other insurers (F52) |
| `refund` | reason (OVERPAYMENT / ENDORSEMENT_CREDIT / CANCELLATION), amount_cents, method (CARD_ORIGINAL / CHECK / INVOICE_CLOUD), **authorisation (by, at)**, funds_cleared_at, status | Released only after funds clear and, above the threshold, after accounting authorises (F54) |
| `write_off`, `followup_activity`, `collection_case` | As in v1 | Collections is Confirm (F60) |
| `commission_entry`, `commission_statement`, `commission_disbursement` | basis (WRITTEN / CASH_RECEIVED), agency hierarchy ref, ACH file id, clearance status | F58, F59, F41 |
| `billing_note` | policy_id, term_id, author, visibility (INTERNAL / AGENT), text | **Shared with Policy, never with Claims** (F72). Ownership: B-21 |
| `billing_outbox`, `audit_entry` | Plus reclassification audit when state, carrier or class changes (F61) | |

The first migration is `0001_init.sql` (P0). Later migrations are additive only.

---

## 5. Domain logic (pure functions, golden-file tested)

The v1 files are kept: `payplan_eligibility.go`, `payplan_switch.go`, `payplan_reassign.go`, `schedule_dates.go`, `schedule_amounts.go`, `fees.go`, `invoice_pipeline.go`, `allocation.go`, `delinquency.go`, `reinstatement.go` and `revenue_split.go`. The new or changed files are:

| File | Responsibility | Feature |
|---|---|---|
| `renewal_establishment.go` | Map the committed term snapshot to a billing term. Carry over the plan and payer. Build bill items including FHCF/FIGA and the policy fee | F34, F36, F45 |
| `fee_distribution.go` | Spread fees and assessments over the instalment schedule according to the extracted rules | F45 |
| `plan_options.go` | The other eligible plans and their instalment breakdowns, for the invoice | F46 |
| `endorsement_spread.go` | Spread the premium delta over the remaining instalments. Custom distributions. Keep the equity calculation correct | F47, F55 |
| `equity.go` | Modified equity earnings. Equity date. Prior-term balance due before the deadline | F38 |
| `lockbox_routing.go` | Route each item to Duck Creek, Siddhi, house account or research | F51, F52 |
| `payment_corrections.go` | Reverse, transfer or refund rules | F53 |
| `refund_release.go` | Funds cleared + authorisation threshold → can release | F54 |
| `nonpay_renewal.go` | Unpaid renewal invoice → flat cancellation date and notices (rules extracted from Duck Creek) | F70 |
| `commission.go` | Rate by policy effective date, written vs. cash basis, parent-child netting, exceptions, transfer dating | F39–F42, F58 |
| `receivables.go` | Policy-level receivables and aging, reconciled to written premium, fees, assessments, endorsements and cash | F61 |

---

## 6. Temporal workflows (`wfc/workflows/billing/`)

| Workflow | ID | Purpose | Feature |
|---|---|---|---|
| **EstablishRenewalBilling** (activity set, called from the Policy `RenewalWorkflow`) | the renewal workflow's ID | Create or carry over the account, term, payer, plan, bill items and schedule. Compensation: void the billing term | F24 → F34, F36, F45 |
| **BillingAccountWorkflow** | `billing-account-{accountId}` | Account lifecycle. Consumes policy events and payment signals. Continue-as-new per term | all |
| **InvoiceRunWorkflow** | `invoice-run-{date}` (schedule) | Generate invoices with plan options. Deliver through FileSphere by channel. Honour the paperless election date | F46, F47 |
| **HeartlandScheduleSyncWorkflow** | `heartland-sync-{termId}` | Push instalment amount and due date when they change. Cadence per B-26 | F48 |
| **CardPaymentWorkflow** | `payment-{paymentId}` | Heartland capture → webhook or poll → allocate. Never "paid" without confirmation | F50 |
| **LockboxImportWorkflow** | `lockbox-{fileId}` | Parse → route Duck Creek / Siddhi / house / research → allocate → reconcile the file total | F50–F52 |
| **RefundReleaseWorkflow** | `refund-{refundId}` | Wait for funds to clear → **FileSphere authorisation task** if above the threshold → release to the original card, a check or Invoice Cloud | F53, F54, F57 |
| **DailyCashSuspenseReport** | `cash-suspense-{date}` (schedule) | Build the daily report for accounting. Raise research and authorisation tasks in FileSphere | F54 |
| **NonPayRenewalWorkflow** | `nonpay-{termId}` | Renewal invoice unpaid → notices → **flat cancellation** command to Policy. A payment before the deadline withdraws it | F70 |
| **DelinquencyWorkflow** | `delinquency-{termId}` | Mid-term instalment non-payment: PCN, cancellation and reinstatement carried over from Duck Creek | F70 |
| **ReinstatementWorkflow** | `reinstate-{termId}` | Reinstatement period and automatic reinstatement when the amount is covered | F70 |
| **EquityBillingWorkflow** | `equity-{termId}` | Before the equity-dating deadline, bill any prior-term balance (coordinated with Duck Creek for migrated prior terms) | F38 |
| **CommissionRunWorkflow** | `commission-{period}` (schedule) | Calculate → statements → ACH disbursement file → consume the clearance file | F58, F59 |
| **ReportingExtractWorkflow** | `billing-extract-{date}` (schedule) | Daily copy of accounting data to the reporting DB for Safepoint's warehouse | F44 |
| **GLPostingWorkflow** | `gl-{period}` | Map to Workday postings (confirm) | F62 |
| CollectionsWorkflow | `collections-{accountId}` | Only if F60 is confirmed | F60 |

**Removed from v1:** RecurringDraftWorkflow, because Heartland collects recurring payments (F56). The generic in-app ApprovalWorkflow is replaced by **FileSphere task triggers** (B-17).

The workflow rules from C§5 still apply. Anything that needs a human creates a **FileSphere task** and waits for a signal with its outcome. Billing never runs its own queue.

---

## 7. Integrations (`integ`)

| Adapter | Direction | Used for | If it's down | Feature |
|---|---|---|---|---|
| **Heartland** (card) | Out / in | One-time card capture (portal and internal) through hosted fields, refunds to the original card, webhooks, **schedule feed** | Retry, then refer. Never mark paid without confirmation | F48, F50, F53 |
| **RT Lawrence lockbox** | In (files) | Check payments from insureds and mortgagees. **Split between Duck Creek and Siddhi** | File-level retry. Unmatched items go to research. The file total must reconcile | F50–F52 |
| ACH / premium finance | In | ACH and PF remittances | Format and vendor TBD (B-09) | F50 |
| **FileSphere** | Out / in | Document storage and history (invoices, notices, statements). **Task triggers** and status callbacks | Retry. Documents queue until it's back | F46, F54, F70 |
| Mortgagee electronic delivery (with Policy, Stage 4) | Out | Mortgagee invoices, with a transmission timestamp | Retry, then fallback | F29, F34 |
| **Commission ACH / clearance files** (accounting) | Out / in | ACH disbursement file (banking details stay in accounting). Clearance-status file | Retry, then alert | F58 |
| **Workday** | Out | GL postings (go-live about January 2027, integration January–April) | Batch retry | F62 |
| Reporting DB | Out | Daily accounting replica | Retry, then alert | F44 |
| Invoice Cloud | Out | Refunds by email or text | Confirm | F57 |
| Collections agency | Out | Referrals | Confirm | F60 |

---

## 8. APIs

### 8.1 Internal BFF routes (`/v1`, `Idempotency-Key` on every write)

| Route | Purpose | Feature |
|---|---|---|
| `GET /v1/billing/accounts/{id}` | Account workspace (payer history, terms, schedule, invoices, payments, notes, activity) | — |
| `GET /v1/billing/accounts/{id}/view?mode=agent\|accounting` | **Simplified agent view** (balance, next instalment, amount due) vs. **detailed accounting view**. The role is enforced on the server | F67 |
| `POST /v1/billing/terms/{id}/bill-to` | Billing-only payer change ("stop billing my mortgage"). Mortgagee changes themselves come from Policy | F34, F37 |
| `GET /v1/billing/terms/{id}/plan-options` · `POST …/plan-change` | Eligible plans with breakdowns. Plan change validated on the server | F36, F46 |
| `GET /v1/billing/invoices/{id}` (+ `/versions`) | An invoice and the invoices it superseded | F47 |
| `POST /v1/billing/accounts/{id}/payments` | Take a card payment through Heartland hosted fields | F50 |
| `POST /v1/billing/payments/{id}/reverse` · `/transfer` · `/refund` | Payment corrections | F53 |
| `GET /v1/billing/reports/cash-suspense?date=` | The daily cash-and-suspense report | F54 |
| `POST /v1/billing/refunds/{id}/authorisation` | Accounting authorises a refund (the FileSphere task links here) | F54 |
| `GET /v1/billing/lockbox/files/{id}` | File routing and reconciliation view | F51 |
| `GET /v1/billing/commission/statements?agency=` (+ `.pdf`, `.csv`) | Statements with a **real CSV/Excel export** | F59 |
| `GET /v1/billing/reports/receivables?asOf=` · `/commission-payable` | Receivables and aging, commission payable/receivable | F61 |
| `GET/POST /v1/billing/notes` | Shared policy and billing notes (per B-21) | F72 |
| `GET/POST /v1/admin/billing/config/{table}/versions` (+ approve, publish) | Config versions with four-eyes approval | — |

**Removed from v1:** `POST /v1/quotes/{id}/payments` and the new-business pay-plan route, because there is no NB in Phase 1. Suspense and approval **worklist** routes are removed because queues live in FileSphere.

### 8.2 Portal routes (`/v1/portal/billing/…`, scoped on the server by the caller's identity)

| Route | Purpose |
|---|---|
| `GET …/summary` | Amount due, next due date, status, alerts |
| `GET …/accounts/{id}/invoices` (+ document from FileSphere) · `…/schedule` | Invoices **with plan options** (F46) and upcoming instalments |
| `POST …/accounts/{id}/payment-sessions` | One-time Heartland payment |
| `GET …/accounts/{id}/payments` (+ receipt) | History and receipts |
| `GET …/accounts/{id}/pay-plans` · `POST …/plan-change-requests` | Plan options and a request to change plan |
| `GET …/accounts/{id}/notices` | Past-due and cancellation notices, with the pay-to-cure amount and deadline |
| `GET …/agent/clients?status=` · `POST …/agent/clients/{id}/payments` | Agent book billing (the simplified view, F67) |
| `GET …/agent/commission-statements` (+ CSV) | Agent commission statements (F59). The parent sees the rolled-up statement; children don't see the parent's rate (F41) |

**Removed from v1:** `PUT …/autopay` (F56 is out of scope). The status of `PUT …/payment-method` depends on B-27, because card-on-file for recurring payments is likely held by Heartland.

### 8.3 Contracts with Policy

- **Inbound activity:** `EstablishRenewalBilling(termSnapshot)`.
- **Inbound events:** `policy.endorsed`, `policy.mortgagee_changed`, `policy.cancelled`, `policy.reinstated`.
- **Outbound events:** `billing.account.status_changed`, `billing.renewal_invoice.unpaid`, `billing.payment.*`, `billing.invoice.issued`.
- **Commands to Policy:** `RequestFlatNonPayCancellation`, `WithdrawCancellation`, `RequestReinstatement`, `RecordPCN`.

---

## 9. Screens

### 9.1 Internal (`<billing-app>`, Acme composite patterns)

| # | Screen | Pattern | Personas | Phase |
|---|---|---|---|---|
| 1 | Billing tab in Policy 360: **agent view and accounting view** | Detail tab | Agent, CSR, Accounting | P1 |
| 2 | Billing Account Workspace (payer history, terms, schedule, invoices, payments, notes, audit) | Detail/Case + rail | CSR, Accounting | P1–P2 |
| 3 | Invoice & Schedule, with plan options and superseded versions | Financial/Estimate | CSR, Agent | P2 |
| 4 | Take Payment (Heartland) and Payment Corrections (reverse / transfer / refund) | Modal / drawer | CSR, Accounting | P3 |
| 5 | Lockbox File view: routing, reconciliation, research items | Detail + table | Accounting | P3 |
| 6 | Daily Cash & Suspense Report + Refund Authorisation (reached from the FileSphere task) | Report + HITL decision modal | Accounting / Treasury | P4 |
| 7 | Non-pay & Cancellation Monitor (renewal non-pay, PCN, mailed dates) | Queue-style *view* (the tasks are in FileSphere) | CSR, Accounting | P5 |
| 8 | Commission Statements & Disbursement (PDF, CSV, ACH file status, clearance) | Queue + detail | Accounting, Commission | P7 |
| 9 | Receivables Aging & Reconciliation reports | Dashboard / report | Accounting | P8 |
| 10 | Billing Config Manager (versions, four-eyes) | Queue + detail | Admin | Day 2. Seed migrations until then |
| 11 | Persona dashboards (A§7) | Dashboard | All | Day 2 |

### 9.2 Portal (ours)

| # | Screen | Audience | Phase |
|---|---|---|---|
| P-1 | Billing Summary | Customer | P2 |
| P-2 | Invoices & Schedule, with plan options | Customer | P2 |
| P-3 | Pay Now (Heartland) + receipt | Customer | P3 |
| P-4 | Payment History & Receipts | Customer | P3 |
| P-5 | Pay-Plan Change request | Customer, Agent | P3 |
| P-6 | Past-Due & Cancellation Notices with pay-to-cure | Customer | P5 |
| P-7 | Agent Book Billing (simplified view, take a payment) | Agent | P3 |
| P-8 | Agent Commission Statements (PDF/CSV) | Agent | P7 |
| P-9 | Customer and agent billing dashboards | Customer, Agent | Day 2 |

**Removed from v1:** Pay at bind and AutoPay enrolment.

---

## 10. Delivery plan (re-based)

### 10.1 Timeline logic

The Siddhi legend defines **Day 1 as "needed for the first renewal offer (April 2027)"**, and the build window is about six months (October 2026 to March 2027). Not every billing capability is used on the day of the first offer, though. The table below gives the **latest operational need-by** for each phase, inferred from the renewal timeline. **B-18 asks the client to confirm** whether Day 1 means "everything live in April" or "live before first use".

| Event | Approx. date | Billing needs |
|---|---|---|
| First renewals committed and offers mailed (≥45 days before effective) | April 2027 | Establishment, renewal invoice with plan options, documents via FileSphere, reporting extract |
| First renewal payments arrive (lockbox and card) | April–May 2027 | Payment intake, lockbox split, allocation, research, corrections |
| First month-end with Siddhi activity | End of April / May 2027 | Receivables reconciliation, commission calculation, statements and ACH file, daily cash and suspense, refunds |
| Equity-dating deadline for prior-term balances | Before each renewal is billed to a mortgagee | Equity billing (F38): **critical path, needs a spike in P0** |
| First renewed terms take effect | ~July 2027 | Flat non-pay cancellation, lapse and reinstatement. Endorsement recalculation on live terms |
| Workday GL | Go-live ~January 2027; integration January–April 2027 | GL posting mapping |

### 10.2 Parallel tracks

To fit the window, the work runs as **four parallel tracks** that share P0:

- **Track A, Account & Invoice:** P1 → P2 → P5 → P6
- **Track B, Payments:** P3 → P4
- **Track C, Commission & Finance:** P7 → P8
- **Track D, Portal:** follows A and B (P-screens in §9.2)

Each phase ends with an **exit gate**: a scenario that passes end to end on the cluster using **migrated FL Manatee renewal data**.

### P0: Foundations & rule extraction (starts immediately, prerequisite for all tracks)
- **Build:**
  - ✅ Module scaffold `modules/billing/` exists and is wired in the app `mycelfile` (2026-09-30). Still to do: `0001_init.sql` (§4) with the MIE tenant.
  - ✅ Sidebar and empty screens exist (2026-09-30):
    - One "Billing" section declared in `service/logic/nav.go`, with 8 entries: accounts, invoices, payments, lockbox, cash-suspense, nonpay, commission, receivables. There is also a drill-down `account` object page.
    - The pages live in `ui/src/pages/`, registered in `ui/src/lib/views.ts` and `pages/index.ts`. Keep the ids in sync across all three.
    - The policy module also has a "Billing" sidebar group (Renewal Billing), so check how the shell renders the two together. See B-12.
  - The money type, the error envelope and the outbox.
  - **Temporal context from the time-travel headers** (B-24).
  - Config versioning and seed loading.
  - The `wfc/workflows/billing` skeleton.
  - `integ` stubs for Heartland and FileSphere.
  - Internal and portal UI shells (B-16).
- **Rule extraction workstream (with the client):**
  - Extract FL Manatee billing config from Duck Creek: plans, down payments, FHCF/FIGA/policy fee and their spread, allocation order, thresholds, notice and cancellation rules for renewal non-pay, and commission rates.
  - Check each item with the client, then load it as seed config.
- **Spikes:**
  - **Equity earnings method** (F38, B-19).
  - **Billing data conversion from Duck Creek** (plan, payer, prior-term balance, Heartland recurring enrolment continuity) (B-20).
- **Exit gate:** the module runs on the cluster. The extracted FL catalog loads and a client sign-off record exists. Golden tests of instalment split and fee spread pass to the cent. A test date set in the header drives all business dates.
- **Blocked by:** B-04/B-22 (ledger vs. Workday boundary), B-16, B-24.

### P1: Billing Establishment at renewal commit (Track A; need-by April 2027)
- **Build:**
  - `EstablishRenewalBilling` activities called from the Policy `RenewalWorkflow` at F24.
  - `bill_to_assignment` for independent payers (F34).
  - **Plan carried over from Duck Creek** (F36).
  - Bill items with **FHCF, FIGA and policy fee** plus their instalment spread (F45).
  - Revenue split.
  - Producer and agency, and the commission basis captured at the effective date (F40).
  - Billing-only "stop billing my mortgage" vs. a mortgagee change from Policy (F37).
  - Billing tab in Policy 360, agent vs. accounting view (F67).
  - Account Workspace, first cut.
  - The **reporting extract** starts here (F44).
- **Exit gate:**
  - A migrated FL HO3 on 4-pay with a mortgagee on file but the insured paying commits its renewal. Billing creates a term with a **4-pay plan and the insured as payer**, the FHCF/FIGA/fee lines spread per the extracted rules, and a schedule matching Duck Creek to the cent.
  - A second policy billed to its escrow mortgagee gets a mortgagee full-pay term.
  - The daily extract contains both.
- **Blocked by:** Policy F24 contract (B-25), B-20.

### P2: Renewal Invoicing & delivery (Track A; need-by April 2027)
- **Build:**
  - InvoiceRunWorkflow.
  - **Invoice with the actual plan and the other plan options** (F46).
  - Documents stored in **FileSphere**.
  - Channels: insured print, **mortgagee electronic with a timestamp**, paperless by **election date**.
  - HeartlandScheduleSyncWorkflow (F48, cadence per B-26).
  - Recurring-payment notices, if confirmed (F49).
  - Invoice & Schedule screen. **Portal:** P-1 Summary and P-2 Invoices.
- **Exit gate:**
  - The P1 policies get renewal invoices on schedule. The 4-pay invoice shows its instalments **and** the annual and 2-pay alternatives.
  - The mortgagee invoice goes out electronically with a timestamp.
  - A paperless insured gets no print.
  - Heartland receives the instalment amounts and due dates.
- **Blocked by:** FileSphere interface (B-28), B-26.

### P3: Payment intake & application (Track B; need-by April–May 2027)
- **Build:**
  - Heartland card payments (portal and internal).
  - **RT Lawrence lockbox import with the Duck Creek/Siddhi split** (F51).
  - ACH and premium-finance intake.
  - `allocation.go`.
  - Research bucket and **house account** (F52).
  - Multi-policy and cross-company checks.
  - **Reverse / transfer / refund requests** (F53).
  - **Auto 2-pay/4-pay on a qualifying down payment, whoever submitted it** (F36).
  - Screens: Take Payment and Corrections, Lockbox File view. **Portal:** P-3 Pay Now, P-4 History, P-5 Plan Change, P-7 Agent Book Billing.
- **Exit gate:**
  - A lockbox file with Duck Creek and Siddhi items splits correctly and each side reconciles to the file total.
  - One mortgage-company check covering three policies (one for another insurer) allocates two and sends one to the house account.
  - An unmatched check goes to research with a FileSphere task.
  - A misapplied payment transfers directly to the right policy with no reverse-and-reapply.
  - A full-pay renewal that receives a qualifying down payment becomes 4-pay.
- **Blocked by:** routing list (F10), B-09 (ACH/PF formats), lockbox file spec.

### P4: Refunds, suspense review & accounting controls (Track B; need-by first month-end, ~May 2027)
- **Build:**
  - **Daily cash-and-suspense report** (F54).
  - RefundReleaseWorkflow: **funds cleared + accounting authorisation above the threshold** before release.
  - Card refunds to the original card, then the policy-side reversal.
  - Check refunds.
  - Invoice Cloud evaluation (F57).
  - The accounting/treasury role (F74).
  - Screen: Cash & Suspense Report + Refund Authorisation.
- **Exit gate:**
  - An endorsement credit creates an overpayment. It appears on the daily report.
  - A high-value refund can't be released until accounting authorises it through the FileSphere task and the funds have cleared.
  - A card refund goes back to the original card.
- **Blocked by:** B-17 (FileSphere task contract), authorisation threshold value.

### P5: Non-payment, cancellation & reinstatement (Track A; need-by July 2027)
- **Build:**
  - NonPayRenewalWorkflow: **unpaid renewal invoice → notices → flat cancellation** command to Policy (F70). A payment before the deadline withdraws it.
  - DelinquencyWorkflow for mid-term instalment non-payment, and ReinstatementWorkflow, both carried over from Duck Creek (R§6–R§7).
  - Status events to Policy.
  - Non-pay Monitor screen. **Portal:** P-6 Notices with pay-to-cure.
- **Exit gate:**
  - An unpaid renewal invoice leads to notices on the extracted schedule, then a flat cancellation accepted by Policy.
  - A payment one day before the deadline withdraws it.
  - A mid-term missed 4-pay instalment follows the carried-over PCN rules.
  - Reinstatement within the period works automatically when the amount is covered.
- **Blocked by:** B-01, B-02, B-07 (extracted FL rules), B-03.

### P6: Mid-term changes & equity billing (Track A; need-by: equity before the first mortgagee renewal billing, endorsements by July 2027)
- **Build:**
  - **Endorsement recalculation keeping the original invoice** (F47).
  - **Spread across remaining instalments**, with custom distributions (F55).
  - Mid-term payer changes (F37).
  - **Equity billing and prior-term balance** (F38): coordinated with Duck Creek for migrated prior terms.
  - Premium-finance config (F35, if confirmed).
- **Exit gate:**
  - An inspection-credit endorsement after invoicing reduces the remaining instalments evenly, re-issues the invoice and keeps the original.
  - A policy with an open prior-term balance is billed before the equity deadline, and the renewal isn't sent to the mortgagee until the balance is resolved.
  - The equity-based cancellation date stays correct after the spread.
- **Blocked by:** B-19 (equity method), B-20 (prior-term data in Duck Creek).

### P7: Commission (Track C; need-by first commission cycle, ~May 2027)
- **Build:**
  - `commission.go`: **rate by policy effective date** (F40).
  - Written vs. cash basis.
  - Parent-child netting (F41), exceptions (F42) and transfer dating (F39) where confirmed.
  - Hierarchy consumed from the CRM sync (F43/F77).
  - CommissionRunWorkflow: statements (PDF + **real CSV**), **ACH disbursement file**, clearance-file intake (F58, F59).
  - Commission screens. **Portal:** P-8.
- **Exit gate:**
  - A renewal issued before a rate change but effective after it gets the new rate.
  - A parent agency receives one netted payment with child detail, and the child can't see the parent's rate.
  - The ACH file and clearance round-trip reconcile.
- **Blocked by:** B-04 (commission ownership), commission plan extraction, CRM hierarchy feed (Policy/Agency team).

### P8: Financial reporting, reconciliation & GL (Track C; extract from P1, reconciliation by first month-end, GL per the Workday timeline)
- **Build:**
  - The reporting extract completed and validated: which fields are extracts and which are calculated (written, earned and unearned premium, days earned) (F44).
  - Formats comparable to Duck Creek.
  - **Receivables aging and reconciliation to the GL**, commission payable/receivable, and the reclassification audit (F61).
  - Workday posting map, and ISO/FHCF readiness (F62, confirm).
- **Exit gate:**
  - For a month containing both Duck Creek and Siddhi renewals, policy-level receivables reconcile through written premium, fees, assessments, endorsements and cash to the GL, with differences explained.
  - The commission payable report ties to the commission run.
- **Blocked by:** B-22 (Workday interface), reporting DB target.

### Day 2 / Confirm backlog (after the April–July 2027 releases, or when confirmed)
- Collections (F60).
- Invoice Cloud refunds, if chosen (F57).
- Recurring-payment notices (F49), if not done in P2.
- Persona dashboards (internal and portal).
- Billing Config Manager UI.
- Agency buy-sell commission (F39), if not confirmed for Day 1.
- **Billing support for out-of-sequence rerate/replay** (policy OOS in July).
- Late charges and reminders (B-03).

### 10.3 Feature → phase map

| Feature | Phase | | Feature | Phase |
|---|---|---|---|---|
| 34 Bill-to | P1 | | 50 Intake | P3 |
| 35 Premium finance | P6 (confirm) | | 51 Lockbox split | P3 |
| 36 Plan continuity + auto 2/4-pay | P1 + P3 | | 52 Research / house / multi-policy | P3 |
| 37 Mortgage billing change | P1 + P6 | | 53 Corrections | P3 + P4 |
| 38 Equity billing | P0 spike → P6 | | 54 Cash & suspense, authorisation | P4 |
| 39 Transfer commission | P7 (confirm) | | 55 Endorsement spread | P6 |
| 40 Commission dating | P1 (capture) + P7 | | 56 Recurring collection | **Out of scope** |
| 41/42 Parent-child, exceptions | P7 (confirm) | | 57 Invoice Cloud | P4 evaluation / Day 2 |
| 43 Entity hierarchy | P7 (consume) | | 58/59 Commission payout, statements | P7 |
| 44 Reporting foundation | P1 → P8 | | 60 Collections | Day 2 (confirm) |
| 45 Fees & assessments | P0 extract → P1 | | 61 Receivables & commission reconciliation | P8 |
| 46 Plan-accurate invoice | P2 | | 62 GL & regulatory | P8 (confirm) |
| 47 Endorsement recalculation | P6 | | 70 Non-pay flat cancel, reinstatement | P5 |
| 48 Heartland schedule sync | P2 | | 67/72/74 Views, notes, roles | P1 / P4 |
| 49 Recurring-payment notices | P2 (confirm) | | | |

---

## 11. Test strategy

- **Parity with Duck Creek (new):** run migrated FL Manatee renewals through Siddhi and Duck Creek side by side, and match schedules, fee spreads, invoice amounts and cancellation dates to the cent. The golden files are Duck Creek outputs.
- **Golden files, to the cent:**
  - down payments
  - instalment split
  - FHCF/FIGA/fee spread
  - auto 2/4-pay boundary
  - allocation order
  - endorsement spread
  - equity dates
  - commission (rate dating, netting)
  - receivables aging
- **Time travel:** every business-date scenario runs through the header-based temporal context: renewal commit, invoice date, non-pay deadline, equity deadline, month-end.
- **Failure tests:**
  - duplicate Heartland webhook
  - activity retried after capture
  - lockbox file re-delivered (idempotent per file and item)
  - FileSphere down (documents queue)
  - Policy command API down during a cancellation (retry and alert; never skip)
- **Coexistence:** lockbox items for Duck Creek policies never post in Siddhi. Routing-list changes mid-file are handled.
- **Portal security:** customers see only their own accounts, agents only their book, parent/child commission visibility is correct, no internal fields leak, and pages meet WCAG 2.1 AA.

---

## 12. Non-negotiables & edge cases

1. **Never mark a payment paid without confirmation.** Never release a refund before the funds clear and any required authorisation is given.
2. **Revenue and owner segregation:** premium, the MGA fee, FHCF and FIGA never mix.
3. **Duck Creek is authoritative for the expiring term.** Siddhi never posts to a Duck Creek term.
4. **No silent failure. No Billing-owned queues:** anything needing a human becomes a FileSphere task.
5. **Money is exact. Business time comes from the temporal context.**
6. **Config, not code.** The rules are extracted Duck Creek values signed off by the client.
7. **Portal access is scoped on the server.**

**Edge cases to handle:**
- A payment arrives on the flat-cancellation deadline.
- A lockbox check covers both a Duck Creek policy and a Siddhi policy.
- A mortgagee pays the renewal while the insured also pays.
- An endorsement credit lands after the last instalment has been invoiced.
- A policy's prior-term balance is still open in Duck Creek when its renewal is billed.
- The payer switches from insured to mortgagee after invoicing.
- A commission rate changes between issue and effective date.
- A refund is requested while the card payment is still unsettled.
- A policy is re-routed from Siddhi back to Duck Creek.

---

## 13. Open decisions (v2)

| ID | Decision | Blocks | Owner | Status |
|---|---|---|---|---|
| B-01 | Billing sends a cancel/reinstate command to Policy (proposed) or drives the transaction itself | P5 | Billing + Policy | Open |
| B-02 | One delinquency and non-pay status model | P5 | Billing + Policy | Open |
| B-03 | Reminders and late charges | Day 2 | Business | Open |
| B-04 | Commission ownership: Billing vs. the `ledger` module. **Commission is Day 1 (F58)** | P0, P7 | Billing + Ledger owner | **Urgent** |
| B-05 | Renewal plan | — | — | **Resolved:** keep the plan (F36) |
| B-06 | SC/NC instalments | — | — | **Moot for Phase 1** (FL only) |
| B-07 | FL notice, grace and cure days, and flat-cancellation timing, extracted from Duck Creek | P5 | Compliance + client | Open (extraction) |
| B-08 | Bind on an unpaid escrow invoice (new business) | — | — | **Moot for Phase 1** (no NB) |
| B-09 | Vendors | P3 | Integration | **Mostly resolved:** Heartland, RT Lawrence, Workday, FileSphere. ACH/PF formats and collections agency still open |
| B-10 | `bff` repo location and route ownership | P1 | Platform | Open |
| B-11 | Tenant codes | P0 | Policy + Billing | **Reduced:** Phase 1 = MIE only. Other insurers appear only as house-account counterparties |
| B-12 | `billing_queue` / `RenewalBilling.svelte` ownership | P2 | Billing + Policy | Open |
| B-13 | Collections stop threshold | Day 2 | Business | Open (F60 confirm) |
| B-14 | Billing accent colour | P0 | UX | Open |
| B-15 | Duck Creek coexistence | — | — | **Now a design input:** dual running 12–16+ months, routing list (F10), Duck Creek authoritative for the expiring term |
| B-16 | Where the portal UI lives, customer-facing style, customer and agent login | P0 | Portal team (us) + Platform + UX | Open |
| **B-17** | FileSphere task contract for billing work (research, refund authorisation, non-pay follow-up): trigger payload, status callback, links back into our screens | P3, P4 | Billing + FileSphere/Safepoint | **New** |
| **B-18** | Does "Day 1" mean everything live in April 2027, or live before first use (April–July 2027, §10.1)? | Plan | Client + PM | **New** |
| **B-19** | Definition of Safepoint's **modified equity earnings method** and the equity-dating deadline | P6 | Client + Billing | **New, critical path** |
| **B-20** | Billing data migrated at renewal: current plan, payer, prior-term balance, Heartland recurring-enrolment continuity, commission history | P1, P6 | Conversion team + Billing | **New** |
| **B-21** | Who owns shared notes (F72): the Policy module, the Billing module, or a shared notes service | P1 | Policy + Billing | **New** |
| **B-22** | Workday GL interface (format, timing) and whether `ledger` has a Phase 1 role | P8 | Finance + Ledger owner | **New** |
| **B-23** | Reporting replica target and format for Safepoint's warehouse (F44) | P1 | Safepoint data team | **New** |
| **B-24** | Time-travel temporal-context header contract | P0 | Platform | **New** |
| **B-25** | Policy F24 renewal-commit contract (term snapshot fields). Also: does the colleague's HO3 **new-business** plan still apply to Phase 1? | P1 | Policy team | **New** |
| **B-26** | Heartland schedule-sync cadence and interface (F48) | P2 | Safepoint + Billing | **New** |
| **B-27** | Card and bank on file for recurring payments: held by Heartland (Safepoint-run) or managed from our portal? | P3 | Safepoint | **New** |
| **B-28** | FileSphere document interface (send, acknowledge, search, versions) and timing, given its ~December rollout | P2 | Stage 4 team + Safepoint | **New** |

---

## 14. Risks

| Risk | Mitigation |
|---|---|
| **Six-month window, with almost all billing needed by April–May 2027** | Four parallel tracks. Get B-18 answered early. Rule extraction starts in P0, not later |
| **Equity billing (F38) is the most complex item and crosses systems** | P0 spike, B-19 answered by the client early, parity tests against Duck Creek |
| Rule extraction from Duck Creek is slow or disputed | Use the rulebook (R§) as the starting catalog. Get client sign-off per table. Keep the golden files as Duck Creek outputs |
| FileSphere (rollout ~December) isn't ready for documents or tasks | Put an adapter interface in place now. Queue documents and tasks until it's available. Agree B-17/B-28 contracts early |
| Lockbox split errors during coexistence post cash to the wrong system | Reconcile file totals per system. Route unmatched items to research, never to a guess. The routing list is versioned |
| Workday go-live (~January) slips | Reporting extract and receivables reconciliation don't depend on the GL. Postings buffer until it's ready |
| Commission duplicated between `ledger` and Billing | Settle B-04 in P0 |
| Scope creep: recurring collection, NB billing, queues | Out-of-scope list in §1. Code review uses the Siddhi scope decisions |

---

## 15. Change log

**v2 (2026-09-30):** re-based on the Project Siddhi Phase 1 backlog.
- Removed the new-business bind slice; renewal commit (F24) is now the trigger.
- Removed recurring drafts and AutoPay (F56 is out of scope).
- Replaced Billing-owned worklists with FileSphere tasks.
- Added F34–F62 items:
  - independent payer, plan carry-over, equity billing, FHCF/FIGA spread;
  - invoice plan options, endorsement spread;
  - lockbox split, house account, direct transfers;
  - cash-and-suspense controls;
  - commission payout, receivables reconciliation, reporting extract, Workday.
- Added flat cancellation when a renewal invoice is unpaid.
- Named the vendors.
- Introduced parallel tracks with need-by dates.
- Decisions resolved or moot: B-05, B-06, B-08 (plus B-09 mostly). New decisions: B-17 to B-28.

**v1 (2026-09-29):** the first draft, assuming new-business billing at bind. It's superseded; a copy is in the session scratchpad only.
