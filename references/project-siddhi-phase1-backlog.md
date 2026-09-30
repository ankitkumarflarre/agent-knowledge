# Reference: Project Siddhi, Phase 1 Feature Backlog (the big picture)

| | |
|---|---|
| Original | [project-siddhi-phase1-backlog-v1.docx](project-siddhi-phase1-backlog-v1.docx) |
| Dated | 2026-09-28 (revised three times after two client backlog reviews on 09/28) |
| Status | **Governing scope document for Phase 1.** Epic/feature level, written to feed Azure DevOps backlog creation. User stories are deliberately not included yet |
| Sources | Safepoint onsite sessions 09/22–09/24, client backlog reviews on 09/28, and cross-checks against the reverse-engineered Duck Creek documentation |

> **Precedence:** where this document conflicts with our earlier notes, this document wins, because it records client-confirmed decisions. The conflicts found so far are in §6.

## 1. Program in one paragraph

**Project Siddhi** replaces Safepoint's Duck Creek **policy and billing** systems. Phase 1 is narrow:
- **Manatee** only, in **Florida**, for **Personal Home & Dwelling**: about 89% of Manatee premium and about 189,000 in-force policies. Mobile home, up to about 20,000 policies, is unconfirmed.
- **Renewal processing only**, not new-business quoting. Policies **migrate into Siddhi at their renewal**.
- The **first renewal offer is targeted for April 2027**. The first renewed population becomes effective around **July 2027**, and out-of-sequence rerate/replay follows in July.
- Migration is continuous and incremental, not a single cutover. Duck Creek and Siddhi will run side by side for **12–16+ months**.
- The stack is confirmed as **Go / Svelte on Azure** (Kubernetes, Temporal, PostgreSQL, messaging, object storage, search and monitoring), with a roughly six-month delivery window.

## 2. Confirmed scope boundaries

| Topic | Decision |
|---|---|
| Writing company / state / product | Manatee / FL / Personal Home & Dwelling. Mobile home: confirm |
| Transactions | Renewal plus servicing of the renewed term. **No new-business quoting** |
| Citizens takeout | In Day-1 scope ("Citizens takeout assumptions"). The post-takeout renewal population in the April release is unconfirmed |
| Non-renewal | Status-based only. A policy already flagged non-renewed or cancelled is excluded at selection. **Once a policy enters Phase 1 renewal, it renews.** Proactive non-renewal is Phase 2 |
| Queues, tasks and referrals | **Owned entirely by FileSphere**, which replaces Image Right from about December. Siddhi only *triggers* submissions into FileSphere and *consumes* the decision or status; **it never manages a queue itself**. The one exception is renewal exceptions, which are resolved by data correction and automatic reprocessing |
| Documents | FileSphere is the document system of record. Siddhi triggers generation and shows searchable history, and doesn't store duplicate files |
| Agency data | One-way sync from the CRM into Siddhi. Nothing flows back. This is a decision, not an open question |
| Claims | A separate system. Only a claims summary (loss type and date) is visible to Siddhi. **Notes are shared between policy and billing only, never with claims** |
| **Billing rule baseline** | **Current Duck Creek billing configuration is the default.** Rules are extracted and validated with the client, not redesigned, unless a specific change is identified |
| Out of Day-1 scope | Advantage and other Manatee lines, new-business quoting, voluntary business and new-business integrations, Louisiana-specific servicing |

**Scope status legend:**
- **Day 1 / Required:** needed for the April 2027 renewal offer.
- **Confirm Priority:** needs a scope decision.
- **Day 2 / Candidate:** a fast-follow within Phase 1.
- **Phase 2 / Deferred.**
- **Out of Project Scope (Safepoint-Internal).**

## 3. The eight lifecycle stages (81 features)

Delivery is organised **by lifecycle stage, not by functional pod**. Enabling platform capabilities are attached to the earliest stage that needs them.

| # | Stage | Business question | Output | Features |
|---|---|---|---|---|
| 1 | Renewal Identification | Which policies enter renewal, and when? | Renewal Candidate Identified | 1–5 |
| 2 | Policy Conversion | Is there a reliable renewal record from Duck Creek? | Renewal-Ready Policy Record | 6–13 |
| 3 | Renewal Evaluation | What is the renewal outcome, and can it run without manual intervention? | Renewal Outcome Determined | 14–23 |
| 4 | Renewal Issuance | How does the renewal become an active term? | Active Renewed Policy | 24–33 |
| **5** | **Billing Establishment** | How is financial responsibility established and maintained? | Billing Relationship Established | **34–45** |
| **6** | **Invoicing** | How are obligations communicated to the right parties? | Financial Obligation Communicated | **46–49** |
| **7** | **Payment Processing** | How are payments received, applied, reconciled and reflected? | Payment Activity Accounted For | **50–62** |
| 8 | Policy Servicing | How is the policy maintained for the rest of the term? | Policy Updated and Serviceable | 63–81 |

The earlier "Issuance Gate" stage was removed. Safepoint has no underwriting gate for renewals ("everything is just being allowed to go through"). Duck Creek's red-stop data-integrity checks, such as more than one first mortgagee, are resolved by fixing the data. The engine then reprocesses the policy on the next cycle.

**Engineering enablement (not business features):**
- The Azure platform: estimated at about $20–30K/month across three environments. It needs dedicated DevOps, and RPO/RTO-aligned backup.
- A **time-travel testing framework**:
  - Each user gets their own temporal context (effective time, business transaction time, GL posting time, system record time), passed in request headers. Testers can pick their own test date without moving the server clock.
  - It is for testing only.

### Non-billing stages (short summary)
- **Stage 1: Renewal Identification**
  - Renewals are selected by status, product, company and date (Day 1).
  - A configurable batch or cycle, with a 60–90-day window. Print-room capacity currently limits this to about one week of renewals per cycle.
  - FL statutory notice window: mail no later than about 45 days before renewal (Day 1).
  - A catastrophe moratorium at renewal needs a priority decision.
- **Stage 2: Policy Conversion**
  - A typed entity model replaces the Duck Creek XML blobs.
  - A migration dashboard and incremental loads into QA.
  - Conversion exceptions go to a Duck Creek-side report or worklist.
  - A **cross-system routing** list records which system owns each policy, keeping the same policy numbers.
  - The Citizens takeout import pipeline.
  - Claims history carryover.
  - Conversion reconciliation, and prior-term true-up / delta processing, both need a priority decision.
- **Stage 3: Renewal Evaluation**
  - An automated renewal engine with an exception worklist and reprocessing.
  - Inflation-guard adjustment to Coverage A (it can be hard-coded on Day 1).
  - Rate capping: 15% the first year, then 10%.
  - Versioned rate books.
  - Consent-to-rate continuity.
  - Priority decisions needed: a configurable rating platform, and rules for which external data to refresh at renewal.
- **Stage 4: Renewal Issuance**
  - **Renewal commit and term activation (24) is the event that starts Billing Establishment.**
  - Document generation happens at commit, and **excludes the invoice**, which belongs to Stage 6.
  - Form-version print suppression and package completeness.
  - Recipient-specific channels: insured copy printed, mortgagee copy electronic with a timestamp, agency notices batched or via IVANS.
  - Print standardisation, with dynamic branding by company and state.
  - **Paperless delivery is based on the election date.**
  - Mailing timed to meet statute.
  - FileSphere integration (priority decision).
  - Subscriber-agreement e-sign (priority decision).
- **Stage 8: Policy Servicing**
  - Search, landing page, alerts and flags.
  - Transaction history with comparison.
  - **Role-based views**, including a *simplified agent billing view* (balance, next instalment date, amount due) separate from a *detailed accounting view*, plus "work on behalf of".
  - Party and mortgagee management, including a master mortgagee list and bulk mortgagee address updates.
  - Endorsements: agent endorsements go to a FileSphere review queue.
  - **Cancellation, lapse and reinstatement** carried forward from Duck Creek (70).
  - Out-of-sequence rerate/replay is Day 2, targeted for July.
  - **Shared policy and billing notes** (72).
  - **Authority and audit roles**, including a separate *accounting/treasury role* with deep billing access but no policy editing (74).
  - Agency management, with buy-sell transfers.
  - Document access and regeneration.
  - Agent self-service is endorsement-only on Day 1.

## 4. Billing stages in detail (our scope)

### Stage 5: Billing Establishment
| # | Feature | Scope | Key points |
|---|---|---|---|
| 34 | Independent bill-to / financial responsibility | **Day 1** | Having a mortgagee or premium-finance company on the policy is separate from who pays. The payer (insured, mortgagee or PF company) is chosen independently at bind, renewal and changes |
| 35 | Premium finance configuration | Confirm | Enable by state, MGA or other axis, including refund and payment-routing rules |
| 36 | **Renewal payment-plan continuity** | **Day 1** | **Keep the insured's plan at renewal; don't reset it to full pay.** Automatically assign 2-pay or 4-pay when a qualifying down payment arrives, **whoever submitted it** (today this depends on the submitter's role) |
| 37 | Mortgage billing relationship change | **Day 1** | A mortgagee change comes from Policy, with an effective date and audit history. "Stop billing my mortgage" is a **billing-only** change, not an endorsement |
| 38 | **Equity billing & prior-term balance** | **Day 1** | Bill any remaining balance before the equity-dating deadline (about 20 days before), so a prior-term balance isn't lost across terms. Implement Safepoint's **modified equity earnings method**. **The most complex billing item** |
| 39 | Agency transfer / buy-sell commission dating | Confirm | New agency earns from the transfer date. Clawbacks go to the acquiring agency. Prior commissions stay with the seller (for 1099) |
| 40 | Commission rate & plan effective-dating | **Day 1** | The rate is chosen by the **policy effective date** and the plan's effective date, not the issue date |
| 41 | Parent-child agency commission | Confirm | By default the parent is paid for all subordinates, with positive and negative amounts netted and supporting detail. Children can optionally be paid individually. Children don't see the parent's rate |
| 42 | Special commission rate exceptions | Confirm | By agency or risk origin, for example takeout groups |
| 43 | Multi-entity hierarchy (MGA → Company → Agency) | Confirm | Configuration-driven. Top-level MGAs are Safepoint MGA and Insight Risk Solutions. Extensible beyond Manatee |
| 44 | **Billing & financial reporting data foundation** | **Day 1** | Define the KPIs first, then persist the data Safepoint needs for **its own warehouse** (the warehouse is out of scope). Replicate accounting data **daily** to a reporting DB. Keep existing billing reports working. Keep formats comparable across Duck Creek and Siddhi policies |
| 45 | **Statutory fees, assessments & allocation** | **Day 1** | FL line items: Hurricane Catastrophe Fund (FHCF) emergency assessment, FIGA assessment, policy fee. Also the rules for spreading them across instalments. **Extracted from Duck Creek config** |

### Stage 6: Invoicing
| # | Feature | Scope | Key points |
|---|---|---|---|
| 46 | **Payment-plan-accurate invoice** | **Day 1** | The invoice shows the insured's actual plan and its instalments, **and the other available plan options** |
| 47 | Post-endorsement invoice recalculation | **Day 1** | Recalculate promptly on endorsement increases and credits (including inspection credits). **Keep the original invoice for audit** |
| 48 | Recurring payment billing sync | Confirm | **Billing is the source of truth** for the instalment amount and due date sent to **Heartland**. The refresh cadence after endorsements needs confirming |
| 49 | Recurring payment compliance communications | Confirm | Notices, receipts, amount-change notices and notice periods. **Evidence gap: a dedicated invoicing session is recommended** |

### Stage 7: Payment Processing
| # | Feature | Scope | Key points |
|---|---|---|---|
| 50 | Multi-channel payment intake | **Day 1** | Checks via the **RT Lawrence lockbox**, from insureds and mortgagees. Cards via **Heartland**, through the portal or the policy system. ACH and premium-finance payments |
| 51 | **Dual-system lockbox routing** | **Day 1** | Split lockbox files between Duck Creek and Siddhi policies while keeping reconciliation intact. Depends on cross-system routing (feature 10) |
| 52 | Unidentified & multi-policy payments | **Day 1** | Unmatched payments go to a research bucket (never refund to a zero-balance old policy). One check can pay many policies, **across writing companies**. Mortgage-company checks that cover other insurers go to a **house account**, and individual checks are issued back once cleared |
| 53 | Payment corrections | **Day 1** | Reverse, transfer or refund a misapplied payment. Heartland card refunds go to the original card, then a policy-side reversal. **Transfer directly between policies** without a reverse-and-reapply |
| 54 | Suspense & overpayment review | **Day 1** | A **daily cash-and-suspense report** (overpayments, endorsement credits, cancellations) is reviewed by accounting **before** an automatic refund is released (after funds clear). **Accounting authorises high-value items before release** (reversing today's order) |
| 55 | Instalment allocation strategy | **Day 1** | **Spread endorsement premium across the remaining instalments** by default, not an immediate catch-up, while keeping equity-based cancellation correct. Allow redistribution when the default splits (e.g. 40/20/20/20) don't apply |
| 56 | Recurring payment enablement | **Out of project scope** | Safepoint runs recurring collection **directly with Heartland**. Siddhi only feeds the instalment amount and due date |
| 57 | Outbound refund channel (Invoice Cloud) | Confirm | Email or text refunds as an alternative to checks |
| 58 | **Commission calculation & disbursement** | **Day 1** | By agent and policy, using written premium **and cash received**. Produce an **ACH disbursement file** like today's check file. Banking details stay in accounting. A returned clearance-status file comes back |
| 59 | **Commission statements** | **Day 1** | Notify the commission contact. PDF view/print and a **real Excel/CSV export** (today's Excel option produces a PDF). Role-based access across production, accounting and commission |
| 60 | Collections workflow | Confirm | Staged notices, escalation and agency referral for unpaid unearned premium, as today |
| 61 | **Receivables & commission reconciliation** | **Day 1** | Policy-level premiums receivable and aging, reconciled through written premium, fees, assessments, endorsements and cash to the GL. Month-end differences today are about $70–100K. A commission payable/receivable report by agent. An audit trail for reclassification when state, carrier or class changes |
| 62 | GL & regulatory reporting readiness | Confirm | Map to **Workday** GL (go-live about January; integration January–April). **ISO** reporting within about 45 days of quarter close, combining Duck Creek and Siddhi data. **FHCF** reporting |

### Stage 8 features that land on Billing
- **70: Cancellation, lapse & reinstatement (Day 1).** Carry forward Duck Creek's cancellation types, reason codes, PCN processing, lapse, reinstatement periods, and short-rate/pro-rata calculation.
  - **An unpaid renewal invoice leads to a FLAT cancellation for non-payment** (not pro-rata or short-rate).
  - The timing, grace period and notice rules come from Duck Creek.
- **72:** shared policy and billing notes.
- **67:** the agent billing view and the accounting view.
- **74:** the accounting/treasury role.

## 5. Cross-stage dependencies that involve Billing
- **Stage 4 → 5/6:** renewal commit (24) triggers billing establishment and invoicing. The invoice isn't part of the Stage 4 documents.
- **Stage 5 → 6:** bill-to (34), plan continuity (36) and fees (45) decide what the invoice shows and who gets it.
- **Stage 5 → 7:** commission setup (39–43) feeds commission payout (58). The reporting foundation (44) feeds reconciliation (61) and regulatory reporting (62).
- **Stage 6 → 7:** recurring sync (48) feeds Heartland. Collection itself is out of scope (56).
- **Stage 8 → 6:** endorsements trigger invoice recalculation (47).
- **Stage 6/7 → 8:** an unpaid renewal invoice triggers flat cancellation (70).
- **Stage 8 → 5:** agency transfers (77) feed commission dating (39). The CRM feeds agency data one way.
- **Stage 2 → 7:** cross-system routing (10) is needed for lockbox splitting (51).

## 6. Impact on our Billing knowledge base (conflicts to resolve)

> **Status 2026-09-30:** S1–S13 are addressed in [plan v2](../plan/billing-implementation-plan.md). The remaining unknowns are tracked there as B-17…B-28.

| # | Earlier assumption (our docs) | What Siddhi says | Affected |
|---|---|---|---|
| S1 | Plan P1 = **new-business pay-in-full at bind**, aligned to the Policy team's HO3 new-business plan | **Phase 1 is renewal-only. New-business quoting is out of Day-1 scope.** The first billing trigger is **renewal commit (24)** | Plan §1, §2, §10 (P1 needs re-basing); the colleague's HO3 plan may be outside Siddhi Phase 1 (confirm) |
| S2 | We build recurring EFT/card drafts and portal **AutoPay enrolment** | **Out of project scope (56).** Heartland runs recurring collection. We only **feed instalment amount and due date** (48) | Plan: RecurringDraftWorkflow, the portal AutoPay route |
| S3 | Renewal forced to Annual? (open question B-05) | **Resolved:** keep the plan at renewal (36) | Rulebook §2.5, plan B-05 |
| S4 | Billing-owned worklists (suspense, approvals, collections) as our own queues | **FileSphere owns all queues and tasks.** Siddhi only triggers and consumes. Suspense review is a **daily report plus accounting authorisation** (54) | Plan §9 screens, ApprovalWorkflow; confirm whether billing queues are also FileSphere tasks |
| S5 | Delinquency: PCN → NOC → non-pay cancellation for all cases | **An unpaid renewal invoice leads to a flat cancellation** (70). Other cancellation types carry over from Duck Creek | Rulebook §6, plan P4, B-02 |
| S6 | Commission ownership open (B-04), later phase | **Commission calculation, ACH disbursement file and statements are Day 1** (58, 59). Commission dated by the policy effective date (40). The entity hierarchy is needed (43) | B-04 is now urgent. Plan P6 moves earlier |
| S7 | GL posting via the `ledger` module | GL is **Workday**. Reporting uses a **daily replica for Safepoint's warehouse** (44). ISO and FHCF reporting (62) | Plan §3, §8.2, B-04 |
| S8 | Tenants MIE and CURE; three companies | **Phase 1 = Manatee (MIE) only**, but one check can span writing companies (52). MGAs: Safepoint MGA and Insight Risk Solutions (43) | B-11, tenant seeds |
| S9 | Vendors unknown (B-09) | **Heartland** (card, replaced Payeezy), **RT Lawrence** (lockbox), **Invoice Cloud** (refund option), **FileSphere** (documents and tasks), **Workday** (GL) | B-09 largely answered |
| S10 | Duck Creek coexistence open (B-15) | Dual running for 12–16+ months, a cross-system routing list, **lockbox file splitting** (51), prior-term true-up (9) | B-15 is now a Day-1 design input |
| S11 | Our own documents and delivery | FileSphere is the document system of record. Paperless is based on the election date. Mortgagee delivery is electronic with a timestamp | Plan §7 |
| S12 | Time and dates in code | A **time-travel test framework** passes the user's temporal context in headers. Billing must never read the server clock directly for business dates | conventions.md |
| S13 | New items not in our plan | Equity billing and the modified equity earnings method (38), independent bill-to (34), billing-only "stop billing mortgage" (37), invoice showing all plan options (46), keeping the original invoice on recalculation (47), house account and cross-company checks (52), direct policy-to-policy transfer (53), spreading endorsement premium across instalments (55), FHCF/FIGA line items (45), shared notes (72), accounting/treasury role (74) | Plan needs new items |

## 7. Open items the document lists
- Is mobile home in Day 1? (Up to about 20,000 policies.)
- Is the Citizens post-takeout renewal population in the April 2027 release?
- Everything else is deferred to the **stage-by-stage detailed-requirements sessions**. For Billing, these include Duck Creek rule extraction, fee amounts, reconciliation checks and flat-cancellation timing. **Stage 6 (Invoicing) needs a dedicated session.**
