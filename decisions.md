# Decisions

<!-- Newest first. Format: ## YYYY-MM-DD — title, then what was decided and why. -->

## 2026-09-30 — Portal billing UI is ours
We are part of the Portal team. The Customer/Agent Portal billing experience (APIs and UI) is in the Billing plan's scope and is not handed off to another team.
Open question B-16: where the portal UI lives, its customer-facing style, and customer/agent login.

## 2026-09-30 — Client-confirmed Phase 1 scope (Project Siddhi backlog, 2026-09-28)
Source: `references/project-siddhi-phase1-backlog.md`. These decisions override earlier assumptions:
- **Scope:** Phase 1 is renewal-only, for Manatee / FL / Home & Dwelling. There is no new-business quoting.
- **Pay plan at renewal:** the insured's plan carries over and isn't reset to full pay or Annual (feature 36). This answers the rulebook's open question in §2.5.
- **Non-payment:** an unpaid renewal invoice leads to a flat cancellation for non-payment (feature 70).
- **Recurring payments:** Safepoint collects them directly with Heartland (out of scope). Billing only sends the instalment amount and due date (features 48, 56).
- **Queues and tasks:** FileSphere owns all of them. Siddhi only triggers submissions and consumes the result.
- **Rules baseline:** Duck Creek's billing configuration is the baseline, validated with the client rather than redesigned.
- **Vendors:** Heartland (card), RT Lawrence (lockbox), Workday (GL), FileSphere (documents and tasks).

## 2026-09-29 — Follow the PAS team's engineering conventions
Billing follows the coding, folder and UI practices from the colleague's PAS End-to-End Implementation Plan, together with the existing `policy` and `ledger` modules. See `conventions.md`.
Billing is built as `app/modules/billing` (its own schema), not as a separate mycel app.

## 2026-09-29 — Duck Creek is a rules source, not a blueprint
Reuse Duck Creek's billing business rules and good practices (pay plans, scheduling, invoicing, PCN/cancellation, collections, reinstatement), so we don't invent them again.
Don't copy its structure (ManuScripts, stored procedures, compressed XML config) and don't copy its known defects. See `architecture/billing-rules-from-duckcreek.md`, §8.

## 2026-09-29 — Team scope is Billing
The user's team works only on the Billing domain of the insurance core platform. Other domains are owned by other teams.
The target design is the reference architecture in `references/insurance-core-architecture.md`.
