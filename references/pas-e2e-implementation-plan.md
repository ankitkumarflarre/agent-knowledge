# Reference: PAS End-to-End Implementation Plan (HO3 New Business)

| | |
|---|---|
| Original | [pas-e2e-implementation-plan.pdf](pas-e2e-implementation-plan.pdf) (28 pages) |
| Author | A colleague on the Policy side (Arshpreet Singh Bhatia), 2026-09-28 |
| Scope of the document | The HO3 new-business journey, from submission to in force, for SPIC in FL |
| **How we use it** | **As the source for coding practices, folder structure and screen patterns**, so that Billing matches the rest of the PAS. It says little about billing. The billing items it does contain are listed in §4 |
| Distilled into | [../conventions.md](../conventions.md) |

## 1. Platform layout it assumes
- **No new mycel app.** New domain logic lands as **modules inside `acme.property.us.policy.app`**. The plan adds `product`, `rules`, `quote`, `rating`, `billing` and `document`, plus the existing `policy` module. Each module has **its own Postgres schema** (named after the module token), and **no module joins across schemas**.
- Every vendor call goes through **`integ`** adapters. Every durable step runs in **`wfc`** (Temporal). **`bff`** only routes, aggregates, rate-limits and dedups idempotency keys.
- **Call path:** Svelte custom element → BFF (`api.lvh.me`) → the module saves draft data and signals the workflow → `wfc` activities call `integ`, `rules`, `rating`, `policy`, `billing` and `document`. The policy module writes the outbox row in the same DB transaction.

## 2. Design rules (the practices we adopt)
1. **Single writer.** Only a `wfc` activity changes lifecycle statuses and writes transactions. HTTP handlers may save draft data but never move a status. The status-writing endpoints accept only the `wfc` service identity.
2. **Rules and money are separate, pure and versioned.** The rules engine decides whether something is allowed and on what terms; it never computes money. Rating computes the price. Neither calls a vendor; both read facts that are already stored.
3. **Facts, not form fields.** Enriched values are stored with their source, evidence level, confidence and fetch time.
4. **Scoped config, never `if state == "FL"`.**
   - All config is keyed by the five-axis key **(carrier_paper, product, state, market_type, peril_scope)** plus effective dates.
   - This applies to rules, coverages, rate tables, tax and fee rows, and templates.
   - Code review rejects hard-coded state checks.
5. **Money is always `BIGINT` cents** (`int64`). Columns are named `*_cents`.
6. **Bi-temporal transactions.** Corrections are made by a reversal plus a reissue, **never by an update or delete**. Compensations use a reversal (`REV`) transaction.
7. **Transactional outbox:** the event row is written in the same DB transaction as the state change.
8. **Immutable, versioned config.**
   - The lifecycle is DRAFT → SIMULATED → APPROVED (four-eyes) → PUBLISHED → RETIRED.
   - A record pins the version that was in force at its creation, except for rules marked "applies immediately".
   - Each evaluation stores the version, a snapshot hash and the actor, so any result can be reproduced.
9. **Explainable and reproducible:** every decision and every number can be traced to its version, its inputs and its correlation ID.
10. **Two views of messages and numbers.** There's an agent/customer view (plain English) and an internal view (thresholds, sources). The server returns only the view the caller's role may see. Premium lines are marked `INTERNAL` or `AGENT`.

## 3. Engineering conventions
- **Migrations:**
  - Each module has `database/migrations/NNNN_description.sql`. A new module starts at `0001_init.sql`.
  - Migrations on existing modules are **additive only**.
  - `schema.sql` is updated **by hand** alongside each migration.
- **Temporal (`wfc`):**
  - Workflow IDs look like `{kind}-{id}`, for example `quote-{id}`, `bind-{id}` and `policy-{id}`.
  - Every vendor call, rule evaluation and clock read happens in an **activity**.
  - Never use `http`, `time.Now()` or `rand` in workflow code. Use `workflow.Now` only for timers.
  - Register everything in `wfc/workflows/register.go`. Set the search attributes `MycelState` and `MycelPhase` at each phase.
  - Everything uses the shared task queue `acme-property-us-policy-tasks` for now.
- **Sagas:** every workflow step has a compensation.
- **Human input** arrives as Temporal signals, for example `uw_decision_recorded` and `payment_result`. Synchronous recalculation uses Temporal **Updates**. Reads use **queries** (`get…State`).
- **Integration adapters:** every adapter implements one contract.
  - It returns a normalised `FactSet {facts[], evidence_level, source_ref, status: OK | PARTIAL | UNAVAILABLE}`. The domain never sees a vendor payload.
  - It has context timeouts, bounded retries with backoff, a circuit breaker, a readiness check and host-bound credentials.
  - Every call is logged to `integ.vendor_call` with `correlation_id` set to the workflow ID.
  - Each adapter declares its degradation up front: *Degrade*, *Refer*, *Retry-then-refer* or *Blocking*.
- **Webhooks:**
  - They bypass the BFF and go straight to `integ`.
  - They land in `integ.webhook_inbox`, deduplicated by key.
  - The owning module records them and relays a signal to the workflow.
  - The workflow also polls the status as a backstop.
- **Payments:**
  - Our idempotency key is passed through to the gateway, so a retry can never double-charge.
  - Card data is captured through the gateway's hosted fields and never touches the PAS.
- **BFF API:**
  - Versioned routes (`/v1/...`) with plural resource nouns and actions as sub-resources: `POST /v1/quotes/{id}/accept`, `POST /v1/stipulations/{id}/decision`.
  - `GET /v1/.../{id}/360` and `/workbench`-style **aggregation endpoints**.
  - **Every write requires an `Idempotency-Key`.** The BFF computes nothing.
  - Admin routes live under `/v1/admin/...`, and approving needs a second user.
- **Testing:**
  - Golden-file tests for money, reproduced to the cent.
  - Outage and timeout tests: a missing vendor response must never turn into a pass.
  - Duplicate-webhook tests and activity-retry-after-charge tests.
  - A phase is done when its scenario passes end to end on the cluster (`mycel up --build`), not when the code merges.

## 4. UI practices
- **Svelte 5 runes**, inside the module's custom element. **Call the BFF only.**
- **Reuse the Acme composite patterns** from `app/docs/ACME-DESIGN-SYSTEM.md`:
  - *Queue/Worklist*
  - *Detail/Case* (main area plus a 320px rail)
  - *Review* (AI findings, then a human decision)
  - *Financial/Estimate*
  - *Dashboard*
- **Prefer tabs over new screens.** Fold minor views into tabs on an existing workspace. For example, Billing Account / Invoice Detail becomes a **tab of Policy 360**, and the Payment Plan Configurator becomes wizard tab 6.
- **Defer config screens** to versioned seed migrations plus an import job. That is still configuration, not code.
- Readiness checklists show blocking reasons. `ProvenanceBadge` shows where a value came from. Locked fields show the rule's reason.

## 5. Billing-relevant content (for [billing-domain.md](../architecture/billing-domain.md))
- **Billing is the module `app/billing`.** It's new in phase P0, with its own schema. It owns:
  - billing account
  - payer (insured or mortgagee)
  - payment plan
  - invoice and invoice lines
  - payments and gateway events
  - the fee/premium split
- **Tables in the plan:**
  - `billing_account`: `payer_type`, `payer_party_id`, `plan`.
  - `invoice` and `invoice_line`: `line_type`, `amount_cents`, `revenue_owner`. This records the **policy fee vs. premium split**.
  - `payment`: `amount_cents`, `method`, `gateway_ref`, `status`, `idempotency_key`.
  - `gateway_event`.
- **Billing steps in BindWorkflow:**
  1. `CreateBillingAccount` and `CreateInvoice`. The lines come from the rating result plus `product.tax_fee_rule`, split into premium and policy fee. The compensation is `VoidInvoice`.
  2. `CollectPayment` through `integ`, then wait for the `payment_result` signal. **Never bind unpaid.** The compensation is `RefundPayment`.
  3. **Payment ≠ issuance:** if issuance fails after payment, the policy stays BOUND and the payment is not refunded.
- **Escrow variant:** if the bill-to party is the mortgagee, a `SendMortgageeInvoice` step is used. Binding on an unpaid invoice is allowed only if `bind_on_escrow_invoice` is set. That flag is open decision **D-06** (owner: Billing / Product).
- **Taxes and fees** live in `product.tax_fee_rule`: `code`, basis FLAT or PCT, `amount`, payee (CARRIER / MGA / STATE).
- **BFF route:** `POST /v1/quotes/{id}/payments` returns a hosted-payment session. It depends on the BFF's idempotency dedup store, a phase-0 prerequisite.
- **Screens:** the Billing Account / Invoice Detail is a tab of Policy 360. The payment step is wizard tab 6: full pay now, **instalments later**. Payment Gateway Reconciliation is deferred.
- **Out of scope in that plan:** GL posting (the fee/premium split is recorded but not posted), endorsement, renewal and cancellation.
