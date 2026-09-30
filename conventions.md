# Conventions

These conventions apply to every agent and every team member working on **Billing**, so that code and screens match the rest of the PAS.

**Sources:**
- The colleague's [PAS End-to-End Implementation Plan](references/pas-e2e-implementation-plan.md).
- `acme.property.us.policy.app/docs/MYCEL-ARCHITECTURE.md` §8 ("LLM Development Rules").
- `acme.property.us.policy.app/docs/ACME-DESIGN-SYSTEM.md`.
- The existing `policy` and `ledger` modules. **When in doubt, copy the pattern from those modules rather than inventing a new one.**

## 1. Where code lives
- **Billing is a module in the main app:** `acme.property.us.policy.app/modules/billing/`.
  - **Done 2026-09-30:** the scaffold is at `modules/billing/` and wired in the app `mycelfile`. Don't create a new mycel app.
  - ⚠ **Gotcha:** `mycel add module <name>` also **rewrites `.mycel.infra/docker-compose.yml` into a standalone form**. That drops the cluster wiring (IAM, shared network, OTEL, KPI). To add a module inside this cluster, scaffold it in a scratch copy of the app, copy `modules/<name>/` across, append its `[[compose]]` block to the app `mycelfile`, and let the cluster's `mycel up` regenerate the compose file.
  - The module gets its own Postgres schema, `billing`.
- **Module layout.** This mirrors `modules/ledger/`, which is the most recent example:
  ```
  modules/billing/
    docs/            overview.md, DESIGN.md (decisions), README.md
    module/
      mycelfile      [module]-only
      database/      migrations/0001_init.sql …, schema.sql
      service/       main.go, go.mod, logic/*.go (+ *_test.go next to each file)
      ui/            src/{App.svelte, entry.ts, pages/, lib/{api,ui,features}}
    module_test/     database/schema_test.go, service/api_test.go
  ```
- **Don't hand-edit files that mycel generates:**
  - `infra/docker-compose.yml`
  - module `Dockerfile`s
  - `.mycel.infra/k8s/*`
  - `fabric/docker-compose.yml`

  You *do* edit `service/logic`, `database/migrations`, `ui/src`, `wfc/workflows` and `mycelfile`.
- **Other apps have fixed jobs:**
  - **Workflows and timers** (schedules, delinquency, PCN, recurring EFT) go in `acme.property.us.policy.wfc/workflows/`.
  - **External calls** (payment gateway, ACH/lockbox files, collections agency, print/mail) go in `acme.property.us.policy.integ` adapters.
  - **Client routing and aggregation** goes in the `bff`. Never put business logic there. *(The `bff` repo isn't in this workspace. Ask where it lives before changing routes.)*
- **Existing billing code in the policy module:** `policy/module/service/logic/billing_queue.go` (`GET /m/policy/billing/queue`) and `ui/src/pages/RenewalBilling.svelte` are renewal-billing views. Follow their API conventions. Decide with the Policy team whether they move to the billing module.

## 2. Domain boundaries
- **Don't join across module schemas.** Other domains talk to Billing only through its API or its events, and Billing talks to them the same way.
- **Single writer:** only `wfc` activities change lifecycle statuses. Examples are an invoice being issued or voided, an account becoming delinquent, a payment being settled, and a cancellation request. HTTP handlers may save drafts and config, but never move a status.
- **Transactional outbox:** each domain event is written in the same DB transaction as the state change it describes.
- **Rules vs. money:** eligibility and decision rules live in rule sets. Amounts come from pure, deterministic functions. Neither of them calls a vendor.

## 3. Data and money
- Money is **`BIGINT` / `int64` cents**, and money columns end in `_cents`. Never use floats for money. Rounding follows the rulebook (remainder on the last instalment).
- **Financial records are immutable and append-only.** Correct them with a reversal plus a new entry. Never use UPDATE or DELETE. This matches `ledger_entry` and `policy_transaction`.
- **All config is effective-dated and scoped**:
  - the five-axis key: carrier_paper, product, state, market_type, peril_scope;
  - plus the billing dimensions: pay plan and channel.

  **Never write `if state == "FL"`.** Use a lookup table.
- **Versioned config** follows DRAFT → APPROVED (four-eyes) → PUBLISHED → RETIRED. A record pins the version that was in force when it was created.
- Carry **correlation, account, policy and party IDs** end to end. `correlation_id` is the workflow ID for workflow-driven calls.
- **Migrations:**
  - The first file is `NNNN_description.sql`, starting at `0001_init.sql`.
  - Later migrations are additive only.
  - Update `schema.sql` by hand in the same change.

## 4. Go service
- Business logic lives in `service/logic/`, with **one concern per file**. Start each file with a package comment saying what the file does and doesn't own; see `billing_queue.go`.
- **Error envelope** (copied from `policy/errors.go`, as `ledger` already did):
  `{"error": {"code","message","field","retryable","correlationId","remediation"}}`
- Use structured logging with `log/slog`. Build on the module SDK `github.com/mycelpf/mycel_module_sdk`.
- **API conventions:**
  - Paths are kebab-case and JSON fields are camelCase.
  - `null` means unknown. Never invent `0` or `""`.
  - Change responses by **adding fields only**. Never rename or remove a field.
- **BFF routes:**
  - Use `/v1/<plural-noun>/{id}/<action>`.
  - Aggregation endpoints (like the `…/360` pattern) combine responses and never compute anything.
  - **Every write needs an `Idempotency-Key`.** Payment calls pass it through to the gateway.
- **Two views:** the server returns only what the caller's role may see. For example, internal notes and thresholds aren't sent to customer or agent roles.

## 5. Temporal workflows (`wfc`)
- Workflow code must be deterministic. Vendor calls, DB calls, clock reads and random numbers belong in **activities**. Use `workflow.Now` and `workflow.NewTimer` for time.
- Workflow IDs follow `{kind}-{id}`, for example `billing-account-{id}`, `invoice-run-{date}` and `collections-{accountId}`.
- Every step has a **compensation** (saga). A compensation is a reversal, never a delete.
- **Human decisions come in as signals**, for example write-off approval or a suspense match. Synchronous recalculation uses **Updates**. Reads use **queries**.
- Register everything in `wfc/workflows/register.go`. Set the search attributes `MycelState` and `MycelPhase`. Use the task queue `acme-property-us-policy-tasks`.
- **Never fail silently.** Use bounded retries, then refer the case to a human or send it to a dead-letter queue, and alert (rulebook §1.4).

### Business time (time-travel testing)
Siddhi requires a per-user **temporal context**: effective time, business transaction time, GL posting time and system record time, passed in **request headers**. Each tester can then use their own test date. Billing code must take "now" from that context (and `workflow.Now` in Temporal), **never from the server clock directly**. The header contract isn't designed yet. See the Siddhi backlog, §3.

## 6. Integrations (`integ`)
- Each adapter returns a normalised result with status `OK | PARTIAL | UNAVAILABLE`. The Billing domain never sees a vendor payload.
- Adapters use timeouts, bounded retries with backoff, a circuit breaker and a readiness check. Each adapter declares whether it *degrades*, *refers*, *retries then refers* or *blocks*. For payment capture that means retry, then refer; **never assume the payment succeeded**.
- Every call is logged to `integ.vendor_call` with its correlation ID.
- Inbound webhooks (gateway, chargeback, ACH returns) go into `integ.webhook_inbox`, deduplicated by key. The owning module records them and signals the workflow, which also polls the status as a backstop.
- **Card and bank data never touch the PAS.** Use the gateway's hosted fields and tokens only.

## 7. UI and screens
- Each module's UI is a **Svelte 5 custom element** using runes (`$state`, `$derived`, `$effect`, `$props`, `onclick=`).
  - Receive `token` and `apibase` as props.
  - Call `${apibase}/…` or the BFF only.
  - Never use `window.location` or absolute URLs.
- **Design system:** `app/docs/ACME-DESIGN-SYSTEM.md` is the single source of truth.
  - Use tokens only; never hard-code hex values.
  - Use a dense operational layout.
  - Use the canonical components (`cp-*`, `os-pill`), not the legacy ones.
  - Assemble screens from the existing **composite patterns** (§11) before inventing anything new.
  - Follow the LLM UX rules in §23.
- **Reuse the existing UI kit** in `policy/module/ui/src/lib/ui/`: `PageHeader`, `DataTable`, `StatusPill`, `StatTile`, `Tabs`, `Section`, `Toolbar`, `FacetBar`, `Banner`, `EmptyState`, `DaysLeft`, `Rationale`, `ComparisonRow`.
- **How Billing screens map to the design patterns:**

  | Billing screen | Pattern |
  |---|---|
  | Suspense queue, write-off approvals, collections, CSR cases | Queue/Worklist |
  | Billing account | Detail/Case (main area + 320px rail) |
  | Invoice and instalment schedule | Financial/Estimate |
  | Management and persona KPIs | Dashboard |
  | Approvals (write-off, refund, rate override) | Review / HITL decision modal |

- Show Billing account and invoice details as a **tab of Policy 360**, as the colleague's plan does. The payment-plan picker is part of the quote wizard's payment tab.
- Billing needs **its own domain accent colour** in the module hero (`mod-hero`, design system §4.6). Pick one that no other domain uses and record it in `decisions.md`.
- Always show loading, empty and error states (design system §18). Show why an action is blocked or a field is locked.

## 8. Testing and definition of done
- Unit tests sit next to the code (`logic/*_test.go`). Module-level API and schema tests go in `module_test/`.
- **Golden-file tests for money:**
  - instalment splits
  - fees
  - thresholds
  - auto plan reassignment
  - allocation order

  They must match to the cent.
- **Failure tests:**
  - a duplicate webhook
  - an activity retried after a charge
  - a gateway timeout
  - an ACH return or NSF after allocation
- **Done** means the scenario passes end to end on the cluster (`mycel up --build`), not just that the code merged.
- Every tenant × state × plan combination we support is covered by a regression test (rulebook §1.7).

## 9. Documentation
- Keep `modules/billing/docs/DESIGN.md` for module-level decisions. Keep cross-team decisions in [decisions.md](decisions.md).
- Anything another agent needs to know goes into `agent-knowledge/`. Don't leave it only in chat.
