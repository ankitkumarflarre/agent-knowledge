# Decisions

<!-- Newest first. Format: ## YYYY-MM-DD — title, then what was decided and why. -->

## 2026-09-30 — Portal billing UI is ours
We are part of the Portal team. The Customer/Agent Portal billing experience (APIs and UI) is in the Billing plan's scope and is not handed off to another team.
Open question B-16: where the portal UI lives, its customer-facing style, and customer/agent login.

## 2026-09-29 — Follow the PAS team's engineering conventions
Billing follows the coding, folder and UI practices from the colleague's PAS End-to-End Implementation Plan, together with the existing `policy` and `ledger` modules. See `conventions.md`.
Billing is built as `app/modules/billing` (its own schema), not as a separate mycel app.

## 2026-09-29 — Duck Creek is a rules source, not a blueprint
Reuse Duck Creek's billing business rules and good practices (pay plans, scheduling, invoicing, PCN/cancellation, collections, reinstatement), so we don't invent them again.
Don't copy its structure (ManuScripts, stored procedures, compressed XML config) and don't copy its known defects. See `architecture/billing-rules-from-duckcreek.md`, §8.

## 2026-09-29 — Team scope is Billing
The user's team works only on the Billing domain of the insurance core platform. Other domains are owned by other teams.
The target design is the reference architecture in `references/insurance-core-architecture.md`.
