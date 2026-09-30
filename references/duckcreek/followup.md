# Duck Creek Billing Follow-Ups: Digest

Source: [followup.html](followup.html) (v2, 2026-08-10, code-verified).

## Architecture
- **One dispatcher** routes 5 processor types across 3 flows (Evaluate, Execute, Generate):

  | Processor | Order | Level | Status |
  |---|---|---|---|
  | PCN | 1 | Policy | Live |
  | REIN | 2 | Policy | Live |
  | RMDR | 3 | Account | **Disabled** |
  | LTCH | 4 | Account | **Disabled** |
  | COL | 5 | Policy | Live |

- Custom Safepoint processors: **NOL** (lapse with direct cancellation), **equity cancellation**, **NOCF**, and **MPR** (dormant).
- There is no workflow engine. Scheduling works through `ScheduledActivity` rows whose status moves **P → I → C/X/F** (pending, in progress, completed / cancelled / failed). A Windows "Billing Monitor" service polls these rows and executes them. There are no named batch jobs.

## Notice lifecycle
1. **Trigger:** invoice generated, payment/credit/debit posted, due date changed, or a manual action. Manual actions are Start, Pause, Resume, Restart, Stop, Letter1–3, Task1 and PolicyTransfer.
2. **Dispatch.** The processor must be enabled.
3. **Eligibility.** For PCN this checks hold codes, `PCNAllow`, the invoice range and term membership.
4. **Target date** from the plan tables, adjusted to a non-blackout day.
5. **Select the activity type.**
6. **Suppression checks.**
7. **Choose the template** by company and LOB.
8. **Persist the activity** with status P.
9. **The monitor executes it:** renders a PDF to the print queue, attaches it to the policy, and emails agency mail for NOC and NOL.
10. **Update the status** to C, X or F.
11. **Follow-up action:** cancel, reinstate, or start/step collections.

## PCN (pending cancellation notice)
The PCN starts the legal cancellation clock.

- **Plans** (trigger days after the due date / tolerance $ / tolerance %):

  | Plan | Trigger days | Tolerance $ | Tolerance % | Other |
  |---|---|---|---|---|
  | PCN1 | 0 | 0 | 0 | |
  | PCN2 | 1 | 5 | 3% | |
  | PCN3 | 2 | 10 | 5% | |
  | PCN4 | 5 | 25 | 10% | no equity |
  | SPCN1 | 5 | 20 | 0 | |
  | SPCN2 | 15 | 20 | 0 | |

  Plans also carry `UseEquity`, `ReschedulePCNEvaluation` and `PropagateToRenewal`.
- **Cancel-days notice:** the live plan is `SPCDN1`, with **10 legal days + 5 mailing days, max 365**. There are nine dormant CDN plans ranging from 10 to 30 legal days.
- **Cancel effective date** = the greater of the equity date and the legal-days date.
- **Rescission plans:** tolerance $ / %, with flags to include past-due, account charges and next installment.
- ⚠ **Every one of the 50 states gets the same notice plan (SPCDN1)**, even though the table is keyed by state. This is a compliance risk (R13).
- **Renewal PCN:** if it's the first invoice, there's zero payment and the policy is active, the flow goes to **NOL** instead.

## Cancellation-family notices
- **NOC:** Notice of Cancellation for the insured and the mortgagee. Sent when the PCN legal and mailing days have passed.
- **NOL:** Notice of Lapse. **Billing cancels the policy directly** (Cancel, reason nonPay) through a cross-tier call. It is scheduled for the next processing day that isn't a blackout day. ⚠ There is no failure handling on the cancel call.
- **Equity cancellation:** if the equity date has been reached and the policy is active, it cancels immediately. Otherwise it schedules an ECN activity for the equity date.
- **NOW:** Notice of Withdrawal. Sent when payment arrives before the cancellation effective date and the cancellation is withdrawn.
- **NOCF:** when a partial payment arrives after a PCN or NOL, a **new PCN is issued with a new 15-day cancel date**.
- **Copies to AOIs:**
  - Safepoint sends copies to 10 interest types: 1st–4th mortgagee, additional insured, additional interest, corporation, premium finance, loss payee and trust.
  - Cajun and Manatee send them to **premium finance only**.
  - This is the **only business rule that differs by company**.
- **Template choice** depends on LOB, for example LA dwelling, LA/AL HO Advantage, and TWIA wind-only.
- **Print channels:**
  - Domestic mail.
  - Foreign-address mail.
  - LexisNexis electronic delivery for enrolled mortgagees.
  - A secondary queue for NOL.
  - IVANS PDF download.

## Reminder (RMDR) and late charge (LTCH)
**Both are disabled.**
- The base reminder offsets were Bronze −5, Silver −7, Gold −8 and Platinum −10 days before the due date. The base reminder Execute step is also a stub.
- Late-charge logic is implemented in the base layer but can't be reached, and there are no carrier rates.

## Collections (COL)
- **Entry:** the balance is over the start threshold and the term is cancelled, cancelled for non-pay, or expired.

  | Plan | Days after cancel | Days after expiry | Balance start | Balance stop |
  |---|---|---|---|---|
  | SFPC | 31 | 1 | $5 | $99,999 (effectively never stops) |
  | COL1 | 1 | 1 | $25 | $25 |
  | COL2 | 2 | 5 | $25 | $25 |

- **Cadence:**
  - SFPC: **Letter1 on day 0, Letter2 on day 21**. There is no Letter3.
  - Base COL1: letters on days 0, 14 and 28, plus a task on day 28.
  - Base COL2: letters on days 0 and 14, plus a task on day 14.
- Actions create a collections record and add an internal task. The task assignee and the signatory are hard-coded placeholders.
- There is a guard against creating duplicate collections records.
- Transfers suppress collections.
- The email "from" address differs by company.

## Reinstatement (REIN)
- **Plans:** Bronze 5, Silver 6, Gold 8, Platinum 10 days. The live plan is SPREIN1 with 5 days.
- **Execution** requires invoice count = 1, status = cancelled for non-pay, and the reinstatement amount fully paid. It then sends Reinstate to Policy.
- Otherwise it raises `ReinstateActivityDatePassed`.

## Suppression rules
| Rule | Condition | Applies to |
|---|---|---|
| PCN tolerance | Balance ≤ tolerance $ or ≤ tolerance % × premium | PCN |
| Payment since last PCN | Any payment allocated since the last PCN | PCN (skip it, or redirect to NOL) |
| Suppressed invoice | The invoice was suppressed | Blocks PCN, RMDR, LTCH and REIN, but **not COL** |
| Policy transfer | A transfer is in progress | Blocks COL |
| Collections stop | Balance ≤ stop threshold | COL: exit collections |
| Duplicate guard | A collections record already exists | COL |
| AOI interest-type filter | See the copies-to-AOIs rule above | NOC, NOL, NOW copies |
| Reinstatement amount | Not yet fully paid | REIN |

Duck Creek has **no** do-not-send, invalid-address or communication-preference suppression. That probably belongs in Party.

## Integrations
- To Policy:
  - **Cancel** and **Reinstate** transactions.
  - A **PCN notification** to Policy (`sendPCNRq`). This one has explicit error handling.
  - Documents attached to the policy.
- Agency Mail (HTTP) for NOC and NOL.
- A print vendor, through file drops.
- The forms engine.
- An internal task queue.
- Async retry.

## Key risks (avoid)
- LTCH and RMDR are silently disabled.
- Billing-initiated cancel and reinstate have no failure handling.
- The same notice rules apply in all 50 states.
- Tables are duplicated three ways across PCN, NOL and equity cancellation.
- There are dormant plan rows.
- Values are hard-coded, such as the task assignee and the signatory.
- The company check is inconsistent, and there's an `IsManateeNOW` path bug.
- There is no communication-preference suppression.
