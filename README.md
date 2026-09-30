# Agent Knowledge Base

Shared, tool-agnostic knowledge for every coding agent on this project. Keep files
short and factual; link to existing docs rather than copying them.

| File | Contents |
|---|---|
| [requirements.md](requirements.md) | What we're building and acceptance criteria |
| [conventions.md](conventions.md) | **Where code lives, module layout, data/money rules, Go, Temporal, integ, UI/design system, testing.** Read this before writing code |
| [decisions.md](decisions.md) | Dated log of decisions and why they were made |
| [architecture/](architecture/) | Architecture designs, one file per topic |
| [references/](references/) | Reference documents: a digest of each, plus its original file when one was shared |

## Stored documents

| Doc | Type | Summary |
|---|---|---|
| [plan/billing-implementation-plan.md](plan/billing-implementation-plan.md) | Plan | **End-to-end Billing implementation plan**: data model, workflows, integrations, APIs, screens, phases P0–P7 with exit gates, open decisions B-01…B-15 |
| [architecture/billing-domain.md](architecture/billing-domain.md) | Architecture | **Billing spec**: scope, Policy↔Billing contracts, flow, allocation and delinquency rules, integrations, personas, audit, open questions |
| [architecture/billing-rules-from-duckcreek.md](architecture/billing-rules-from-duckcreek.md) | Architecture | **Billing rulebook**: pay plans, schedule, invoicing, fees, follow-ups (PCN, cancel, lapse, reinstate, collections) carried over from Duck Creek. Marked adopt / config / confirm / avoid |
| [references/duckcreek/](references/duckcreek/README.md) | Reference | Digests and originals of 3 Duck Creek analyses (pay plans, installment/invoice, follow-ups) |
| [references/pas-e2e-implementation-plan.md](references/pas-e2e-implementation-plan.md) | Reference | A colleague's HO3 new-business plan. **Used for coding and UI practices** (distilled into conventions.md), plus the billing items it contains. The original PDF is next to it |
| [references/insurance-core-architecture.md](references/insurance-core-architecture.md) | Reference | Platform-wide domain map, dependencies, guardrails, Forms Engine, downstream data. The original HTML is next to it |

## Existing project docs (source of truth — link, don't copy)

- `acme.property.us.policy.app/docs/` — `OVERVIEW.md`, `MYCEL-ARCHITECTURE.md`,
  `INFRA-CONTEXT.md`, `ACME-DESIGN-SYSTEM.md`, `business-context/`, `modules/`
- `acme.property.us.policy.mycel/docs/` — `requirements/`, `plan/`, `renewal/`, `ux/`, `safepoint/`
- `acme.property.us.policy.mycel/RUNBOOK.md`
- `MYCEL_CLI_USAGE.md`
