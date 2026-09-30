# Billing Rulebook (carried over from Duck Creek)

> **Purpose:** these are the business rules and good practices we reuse from the current Safepoint Duck Creek Billing, so we don't have to invent them again. **This is not a 1:1 replica.** Keep the behaviour, drop the Duck Creek mechanics and defects.
> **Sources:** [references/duckcreek/](../references/duckcreek/README.md) (pay plans, installment/invoice, follow-ups).
> **Companion:** [billing-domain.md](billing-domain.md) (target architecture). Where the two conflict, see §9.
>
> Legend:
> - ✅ **Adopt**: a proven rule or practice.
> - ⚙️ **Adopt as config**: keep the behaviour, but the numbers are data, not code.
> - ❓ **Confirm**: a Safepoint quirk. The business must confirm it before we build it.
> - ⛔ **Avoid**: a known Duck Creek defect or anti-pattern.

## 1. Cross-cutting design principles
1. ✅ **One rule catalog for all companies.** Safepoint, Cajun and Manatee share every money and timing rule. Company differences are presentation and routing only:
   - templates
   - print queue
   - email sender
   - log prefix
   - gateway label
   - the AOI copy filter (§6.4)

   Model the company as a **tenant/brand config**, not as branches in the code.
2. ⚙️ **Everything is table-driven and effective-dated**, keyed by the dimensions it actually varies on: state, LOB, product, plan and channel. This fixes Duck Creek risks R1 and R12.
3. ⛔ **Never enforce eligibility only in the UI.** Validate on the server on every path: API, batch and events. This fixes R7.
4. ⛔ **Never swallow failures.**
   - Every automated plan change, cancel request, reinstate request and notice dispatch has explicit error handling, a retry, a dead-letter queue and an alert.
   - Duck Creek's empty `onFailure` and unguarded cancel/reinstate calls are the main anti-pattern to avoid.
5. ⛔ **One company/tenant identifier.** Resolve it once, in one place. Duck Creek has two identifiers that aren't kept in sync, and the check is duplicated five or more times.
6. ✅ **Scheduled activities with a status machine:** pending → in progress → completed / cancelled / failed. Also support manual pause, resume, restart and stop. *(Recommendation: implement these as Temporal workflows or timers, since `acme.property.us.policy.wfc` already exists, rather than as a polling service.)*
7. ✅ **Every rule change is testable.** Each tenant × state × plan combination needs regression coverage. In Duck Creek, Cajun and Manatee were never tested (R13).

## 2. Pay plans
### 2.1 Structure
- ✅ Use two levels: an **account plan** (billing arrangement or state group) holds a set of valid **policy plans** (payment schedules).
- ⚙️ A policy plan's attributes:
  - frequency (annual, semi-annual, quarterly, monthly, custom)
  - max installments
  - down payment % per transaction type × receivable type
  - commission scheme
  - whether a PCN is allowed
  - paid-in-full discount eligibility
  - follow-up plan
  - offsets: early new-business invoice, early renewal invoice, and last due date before expiry
- ⚙️ The current catalog (seed data) is summarised below. See [payplans.md](../references/duckcreek/payplans.md) for the full table.

  | State group | Plans (down payment %) |
  |---|---|
  | FL | Annual, Semi 60%, Quarterly 40%, 9-Pay 20% (Advantage only) |
  | LA | Annual, Semi 55%, Quarterly 40% |
  | TX | Annual, Semi 60%, Quarterly 40% |
  | LA/AL/TX/MS Advantage | Annual, Semi 55%, Quarterly 31% |
  | SC/NC | Annual only ❓ |
  | Mortgagee/escrow (FL) | Full-pay only |
  | Premium finance (LA/TX/FL) | Full-pay only |

### 2.2 Eligibility
- ✅ The plan must be valid for the account plan. Some plans also have extra gates, such as 9-Pay requiring an Advantage product. Validate on the server.
- ✅ LOB does **not** affect plan availability or the money math. The one exception is the HomeownersAdvantage fee add-on in §2.4.
- ❓ **SC/NC has no installment option.** Is that a regulatory restriction or an unfinished build?

### 2.3 Automatic plan switch on billing-party change
- ✅ When an **endorsement** adds or removes a **mortgagee/escrow** or a **premium finance company**, the account and policy plan switch automatically. Removing the party reverts the plan.

  | Change | Target plan |
  |---|---|
  | Escrow on (FL) | Mortgagee full-pay |
  | Premium finance added (LA/TX/FL) | Premium-finance full-pay |
  | Otherwise | The state's base plan (Advantage or not) |

- ⛔ Implement this as a **lookup table**: (state, product/Advantage, escrow, mortgagee, premium finance) → target plan. Don't use hard-coded branches.

### 2.4 Automatic plan reassignment from the first payment
- ✅ If a **partial first payment** arrives on an Annual plan, re-plan it to Semi-annual or Quarterly when the payment meets that plan's threshold. This also applies to flat-cancelled policies, using the flat-cancellation totals. It runs on single, batch and identified-from-suspense payments.
- ⚙️ **Threshold = Premium × plan % + Tax + Fee + add-on.** The % is 60/55 for semi-annual and 30/31 for quarterly. The add-on depends on the plan group:
  - $13, $6 or $18 depending on the group;
  - for HomeownersAdvantage, MIN(Premium × 0.18, 10);
  - for the Advantage groups, a band by premium: under $500 → $4, $500–649 → $5, $650 and above → $6.
- ✅ Mortgagee and premium-finance plans are excluded.
- ❓ Is the boundary ≥ or >? Duck Creek never confirmed it. Test it explicitly.

### 2.5 Renewal
- ✅ **Resolved by the client (Siddhi F36):** keep the insured's plan at renewal, and don't force it to Annual. The Duck Creek forced-Annual rule below is **not** carried forward.
- ❓ Duck Creek **forces renewals to the Annual plan**, except for mortgagee and premium-finance plans. It matches by caption, which is fragile. Confirm whether the business still wants this, or whether the insured's plan should carry over.
- ✅ Renewal billing config for mortgagee and premium-finance terms is reset rather than carried over.

## 3. Installment schedule
### 3.1 Principles
- ✅ **The schedule (money due) is separate from the invoice (document).** Suppressing an invoice never pauses collections.
- ✅ The schedule is built at two levels: the **term level** (new business, renewal) and the **item level** (endorsements, reinstatement, miscellaneous debits and credits mid-term).
- ✅ **Preview / projected schedule:** a what-if view that changes nothing. Useful for CSRs and underwriters, for example to show the impact before an endorsement is committed.

### 3.2 Dates
- ✅ Due dates run from the start date to the **end date**. The end date is term expiry minus the plan's "last due date before expiry" offset. There are at most `maxInstallments` dates.
- ✅ **Target due day** is clamped to the month length.
- ✅ **Invoice date = due date − channel lead time.** The lead time is set per account and differs for paper, e-invoice and EFT draw. The first installment gets **extra lead days** for new business and for renewal.
- ✅ **Blackout calendar:** shift dates off holidays and non-business days.
- ✅ **Agency-billed and collections-agency accounts** skip both the lead time and the blackout shift.
- ✅ **Due-date precedence**, highest first:
  1. missed/late-entered override
  2. recovery push
  3. early first-invoice offset
  4. channel lead time
  5. due-day clamp
  6. blackout shift
- ✅ **Missed installment:** if an item posts after its natural first due date and after the last scheduled installment, bill it as a catch-up, never as a past-due date.
- ❓ The late-entry and rescission redistribution strategies (`LateEnterStgy` / `RescRdstStgy`) are referenced in Duck Creek but not implemented. We need to define them.
- ❓ There are **no short-term-policy rules**. Short terms fall out of the date window. Decide whether that's enough.

### 3.3 Amounts
- ✅ Down payment first. Then split the remainder evenly, **rounded to cents**, and **put the rounding remainder on the last installment**. Split commission the same way.
- ✅ Zero possible installment dates is an explicit error. Never produce an empty schedule silently.
- ✅ **Recalculate** on endorsement, reinstatement, a pay-plan or frequency change, and miscellaneous debits or credits. Remove stale future installments first.
- ✅ Keep a history of the pay plan and frequency for each term. Duck Creek keeps policy-term history.

## 4. Invoicing
### 4.1 Generation pipeline
1. ✅ **Threshold pre-check.** If the amount is under the channel threshold, the result is `BelowThreshold` and no document is produced.
2. ✅ **Recovery check** (§4.4).
3. ✅ **Suppress-first.** If the account is set to suppress the first invoice, the result is `SuppressFirst`.
4. Gather the invoice data.
5. ✅ **Service charge** by channel, as flat or %, with a min/max collar. ⚙️ There are exclusions by invoice line, receivable and transaction type. Service-charge plans are keyed by LOB × form × state, with tiers by outstanding balance.
6. ✅ **Threshold post-check.** Adding the service charge can lift the invoice over the threshold.
7. Render the invoice through the shared Forms Engine, and attach and dispatch it.

The result is one of `Success`, `BelowThreshold` or `SuppressFirst`. **Suppressed invoices still schedule their follow-ups.**

### 4.2 Channels
- ✅ Each account has an invoice method: paper, EFT (bank) or EFT by card. **E-invoice is a delivery overlay on paper**, not a separate method. Each channel has its own threshold, service charge and lead days.
- ✅ **EFT retry:** a configured retry count and retry days.
- ✅ **Debit change notice:** notify the customer when the EFT draft amount changes by at least a threshold.
- ✅ Handle **e-invoice bounce-back**: undeliverable email triggers a fallback task.
- ✅ **EFT authorisation form required** is a per-account flag.
- ✅ **Roll policies with a PCN into the account total** is a per-account flag.

### 4.3 Fees and thresholds (⚙️ config, current values)

| Item | Current value |
|---|---|
| Invoice threshold | FL/TX $6; LA/AL/MS/SC/NC $7 |
| Installment setup fee | $10, on the first invoice of non-Annual plans (new business and renewal) |
| Previously-billed fee on a frequency change | FL $3, LA $6, TX $8 |
| Reinstatement fee after non-pay cancellation | $50 (SC/NC) |
| Small-balance auto write-off on pro-rata cancellation | ≤ $5 |

Recalculate the invoice threshold on every policy transaction.

### 4.4 Late invoice recovery
- ✅ If an invoice run is late, the recovery plan (⚙️ keyed by plan and collection method) chooses one of:
  - push that due date forward;
  - re-spread the remaining installments;
  - issue a recovery invoice on the original due date.
- ✅ **Regulatory gate:** never change an EFT draft amount or date inside the required advance-notice window. The window has separate offsets for amount changes and date changes.

## 5. Payments, allocation and suspense
The allocation order in [billing-domain.md](billing-domain.md) §4 stays the primary rule. Duck Creek adds these:
- ✅ **Flat cancellation:** write off any open installment setup fee. If a payment is already allocated to it, move the payment to suspense instead.
- ✅ **Pro-rata cancellation:** hold any overpayment and schedule its release (don't refund immediately). Write off dust balances automatically.
- ✅ **Reinstatement:** reverse the write-offs made at cancellation.
- ✅ Suspense counts toward auto-reinstatement (§7.2) and auto plan reassignment (§2.4).

## 6. Delinquency and follow-ups
This replaces the generic "Past Due → Delinquent → Dunning" wording in billing-domain.md with Duck Creek's proven model.

### 6.1 Follow-up engine
- ✅ Follow-up processors: **PCN, Reinstate, Reminder, Late charge and Collections**. Each has Evaluate, Execute and Generate steps, a processing level (policy or account) and an order.
- ✅ **Triggers:**
  - invoice generated or suppressed
  - payment, credit or debit posted
  - due date changed
  - manual actions: start, pause, resume, restart, stop, send letter N, create task, policy transfer
- ✅ Every follow-up is a **scheduled activity with a status**, and every run is written to the audit trail.

### 6.2 PCN (pending cancellation notice)
- ⚙️ **PCN date = due date + trigger days**, moved to a non-blackout day. Plans range from 0 to 15 days. The live plans are 5 and 15.
- ✅ **Tolerance suppression:** no PCN if the unpaid balance is at most the tolerance $ **or** at most the tolerance % × premium.
- ✅ **Payment-since-last-PCN** check: avoids redundant notices.
- ⚙️ **Cancellation effective date** = the greater of:
  - the **equity date** (the date the paid premium covers up to), and
  - PCN + **legal notice days + mailing days**.

  The current values are 10 legal + 5 mailing, capped at 365.
- ⛔ **The notice period is the same in all 50 states in Duck Creek.** We must key it **by state** (and by LOB/product where the law requires), with legal sign-off. Treat this as a compliance requirement.
- ✅ **Rescission plan:** rules for when a cancellation is rescinded (tolerance $ / %, and whether past-due, account charges and next installment count).
- ✅ **Partial payment after a PCN or lapse:** issue a new PCN with a new cancel date (15 days in Duck Creek).
- ✅ **Renewal PCN with zero payment on the first renewal invoice:** go to the **Notice of Lapse** path (§6.3), not the normal non-pay cancellation.

### 6.3 Cancellation-family notices

| Notice | When | Outcome |
|---|---|---|
| **NOC** (insured + mortgagee copy) | The PCN legal and mailing days have passed | Non-pay cancellation |
| **NOL** (lapse) | Renewal not paid | Lapse/cancel for non-pay on the next non-blackout day |
| **Equity cancellation** | The paid-to (equity) date is reached | Cancel now, or schedule the cancellation for the equity date |
| **NOW** (withdrawal) | Payment received before the cancellation effective date | Cancellation withdrawn, and the insured and AOIs notified |

- ✅ Each notice is produced through the shared Forms Engine. The template is chosen by **tenant + state + LOB** (some notice form numbers are LOB-specific) and stored in the DMS.
- ✅ Print channels: domestic mail, foreign-address mail, electronic delivery to mortgagees (LexisNexis), and agent download (IVANS). ✅ The **mailed date is the evidentiary timestamp**. This matches renewal-plan rule D4.

### 6.4 Copies to additional interests (AOI)
- ⚙️ Cancellation, lapse and withdrawal notices are copied to additional interests. Which interest types get a copy is **tenant config**:
  - Safepoint: 1st–4th mortgagee, additional insured, additional interest, corporation, premium finance, loss payee, trust.
  - Cajun and Manatee: premium finance only.
- ❓ Is the narrower list for Cajun and Manatee intentional?

### 6.5 Reminder notices and late charges
- ❓ Both are **disabled** in Duck Creek. The target architecture's "dunning / reminder letter" suggests we want reminders. Decide whether each is in scope.
  - The base reminder offsets were 5, 7, 8 and 10 days before the due date.
  - There are no late-charge rates.

### 6.6 Collections
- ⚙️ **Entry:** the term is cancelled, cancelled for non-pay, or expired, **and** the balance is above the start threshold after N days.
  - Current plan (SFPC): 31 days after cancellation or 1 day after expiry; balance over $5.
- ⚙️ **Letter cadence:** Letter 1 on day 0, Letter 2 on day 21. The base plans also have Letter 3 on day 28 and an internal task.
- ✅ **Exit** when the balance falls to the stop threshold or below. ❓ SFPC's stop threshold of $99,999 means it effectively never exits. Is that intended?
- ✅ Guard against duplicate collection records. Suppress collections while a **policy transfer** is in progress.
- ✅ Collections still run even when the invoice was suppressed.
- ✅ Allow a referral to an external collections agency. This is the collections-agency account type, which skips lead time and blackouts.
- ⛔ Don't hard-code the task assignee or signatory. Use a configurable queue or role.

### 6.7 Suppression rules to add (missing in Duck Creek)
- ✅ Do-not-send, invalid address, and communication preference (paper vs. email). Read these from **Party**, and respect consent. The renewal plan also requires this (D4/D5).

## 7. Billing ↔ Policy interactions
### 7.1 What Billing tells Policy
- PCN issued, with its cancellation effective date.
- Non-pay cancellation requested (NOC).
- Lapse (NOL).
- Equity cancellation.
- Cancellation withdrawn (NOW).
- Reinstatement requested.
- Payment status and delinquency.

### 7.2 Automatic reinstatement
✅ The system reinstates without user action when **all** of these hold:
- the policy was cancelled for non-pay or flat-cancelled after a lapse notice;
- it is within the reinstatement window (⚙️ 5 days in the live plan, 15 days for the post-lapse path);
- the invoice count and status rules pass;
- **the payment or suspense amount covers the amount due.**

If the payment is short, raise a task. Never reinstate partially.

### 7.3 Policy transactions and their billing effect
| Policy transaction | Billing effect |
|---|---|
| New business | Build the schedule. Apply down payment, installment setup fee and first-invoice offset |
| Endorsement | Rebuild at item level. Possible plan switch (§2.3). Previously-billed fee if the frequency changes |
| Renewal | Build the schedule (plan rule in §2.5). Renewal offset. Installment setup fee |
| Flat / pro-rata cancellation | Write off, suspend or hold overpayment. Expire pending activities. No reschedule |
| Reinstatement | Reverse write-offs. Rebuild. Reinstatement fee |
| Restart | Rebuild and expire activities |
| Rescind | Rescission-plan rules |
| Audit | Adjust the existing schedule |
| Paid in full | Apply the paid-in-full discount |
| Information only | No effect |
| ❓ Rewrite / reissue | Not implemented in Duck Creek. Define it |

## 8. Anti-patterns from Duck Creek not to repeat (⛔ checklist)
- [ ] Hard-coded branches instead of lookup tables (endorsement matrix, company checks).
- [ ] Silent failure handlers. Cross-domain calls without error handling.
- [ ] UI-only validation.
- [ ] Rule tables copy-pasted across modules. Keep one source per rule.
- [ ] Dormant or unused configuration rows. Placeholder hard-coded people or phone numbers.
- [ ] Matching on captions or display strings instead of codes.
- [ ] Parsing structured data out of delimited strings. Storing config as compressed XML blobs.
- [ ] Features quietly disabled through a no-op switch (late charge, reminder). Use explicit feature flags that are visible in admin screens.

## 9. Conflicts with the target architecture (need decisions)
1. **Who cancels?** The architecture has Billing send a *non-pay cancellation indicator* and Policy decide. In Duck Creek, Billing **directly starts** Cancel and Reinstate transactions in Policy (NOL, equity cancellation, auto-reinstatement). The guardrail "APIs for commands, events for propagation" suggests Billing calls a **Policy command API** (request cancel / request reinstate) and Policy owns the transaction. This needs to be agreed with the Policy team.
2. **Delinquency model.** The architecture's statuses (Current / Past Due / Delinquent / Written-off) need to map to Duck Creek's PCN → NOC → cancel flow, including equity-based cancellation and lapse. We need one agreed state list and event list.
3. **Reminder and late-charge features.** The architecture shows reminder and dunning letters, but Duck Creek has both disabled. See §6.5.
4. **Commission** is computed on earned premium, and Duck Creek splits commission per installment. The existing `ledger` module also computes commission. Decide which one owns it.
5. **Mortgagee/escrow and premium-finance billing** are central in Duck Creek (dedicated plans and plan switching), but they are absent from the architecture diagram. They are in scope for Billing.
