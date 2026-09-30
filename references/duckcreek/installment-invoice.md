# Duck Creek Installment Scheduling & Invoice Generation: Digest

Source: [installment-invoice.html](installment-invoice.html) (v2.0, code-verified).

## Core principle
**The installment schedule (when money is due) is separate from the invoice (the document).** An installment coming due does not always produce a printed invoice. Invoice generation returns one of these codes:
- `Success`: the invoice was produced.
- `BelowThreshold`: suppressed because the amount is under the threshold.
- `SuppressFirst`: suppressed because it is the first invoice and the account is set to suppress it.

**Suppressed invoices still schedule their follow-ups** (PCN and collections). Suppression stops the document, not the collections lifecycle.

## End-to-end flow
1. A policy transaction sends bill items to Billing.
2. The frequency calculator builds the installment dates.
3. The amount calculator splits the money.
4. The schedule is persisted.
5. The nightly cycle reaches each installment date and generates the invoice.
6. Post-invoice events schedule a late charge, a reminder, a PCN, and cleanup of expired activities.
7. An email event sends the invoice. If the invoice was suppressed, a suppressed-invoice event still schedules the follow-ups.

## Installment date algorithm
- **Start date:** the earlier of the term effective date (or the item effective date for mid-term items) and the processing date.
- **End date:** term expiration minus `LastAvailDueDtDisp`, which is the "last due date before expiry" ceiling.
- **Count:** at most `MaxNumberInstallments`, and no due date after the end date.
- **Target due day** of the month is clamped to the month length. For example, a target of 31 becomes 28 in February.
- **Installment (invoice) date = due date − lead days.** The lead depends on the channel:
  - EFT: `EFTDrawOffsetDays`.
  - Electronic invoice: `ElectronicInvoiceOffsetDays`.
  - Paper: `BillProductionOffsetDays`.
- **Early offsets:** extra lead days on the first new-business invoice and on the first renewal invoice.
- **Blackout calendar:** dates are shifted off blackout days.
- **Agency (`AGT`) and collections-agency (`COL2`) accounts** skip both the lead time and the blackout shift.
- There are two builders: a term-level one for full-term and renewal, and an item-level one for endorsement, reinstatement and miscellaneous items.

## Due-date precedence (highest first)
1. Missed or late-entered override: the item posts after its natural first slot.
2. Invoice-recovery push.
3. Early-invoice offset for new business or renewal (first installment only).
4. Channel lead days.
5. Target due-day clamping.
6. Blackout shift (skipped for AGT and COL2).

## Recalculation
- Endorsements, reinstatements and miscellaneous credits or debits trigger a rebuild. Stale future installments are purged.
- A "missed installment" is flagged when an item's effective date is after its first due date **and** the transaction date is after the last scheduled installment.
- Late-entry and rescission redistribution strategies are referenced but not implemented in the manuscripts.
- **There are no short-term-policy rules.** Short terms fall out of the date window and the max-installments cap.

## Amount split
- Down payment % is keyed by transaction type + receivable type + sub-type, with a wildcard fallback.
- Remaining amount = item amount − down payment.
- Each installment = remaining amount ÷ number of dates, **rounded to cents**, with **the remainder placed on the last installment**.
- Commission is split the same way.
- Zero dates raises the error `NoInstallments`.

## Invoice generation (7 steps)
1. **Threshold pre-check:** amount to invoice vs. threshold. `BelowThreshold` if under.
2. **Schedule-override (recovery) check.**
3. **Suppress-first check:** `SuppressFirst` when the flag is set and it's the first invoice.
4. Gather invoice data.
5. **Service charge:** assessed per channel as flat, %, or with a min/max collar. There are exclusions by invoice line, receivable and transaction type.
6. **Threshold post-check.** Adding the service charge can lift the amount over the threshold.
7. **Print and attach the invoice.** This passes the account, invoice date, due date, and whether the invoice is out of schedule.

## Delivery channels (per account)
- Channels are paper (`PA`), EFT and EFT-by-card (`EFTC`). Electronic invoicing is an overlay on paper (`ElectronicInvoice=1`), not a fourth channel.
- Each channel has its own service charge and threshold. EFT also has a retry count and retry days.
- **Debit change notice:** sent on EFT when the draft amount changes by at least `ChgNotThresh`.
- There is a bounce-back workflow for undeliverable e-invoices.
- Account invoice settings also include:
  - `IncludePCN`: roll the current-due amount of policies with a PCN into the total.
  - `EFTFrmReq`: an EFT authorisation form is required.
  - `PrintJob` and `FormMS`: print routing and invoice template.

## Invoice recovery (custom Safepoint logic for late invoices)
This applies when a schedule-override plan exists and the invoice run is late. Config is looked up per recovery plan and collection method:
- `PushDueDate`
- `PushDueDateNextInvoiceOffsetDays`
- `RecoveryOffsetDays`
- `RedistributionStrategyCode`

Recovery then does one of these:
- Pushes the single due date forward.
- Re-spreads the remaining installments.
- Generates a recovery invoice without moving the due date.

**EFT regulatory gate:** recovery cannot skip the advance-notice windows for EFT amount changes and due-date changes (`EFTAmountChangeOffsetDays`, `EFTDueDateChangeOffsetDays`).

## Transaction impact
| Transaction | Effect |
|---|---|
| NBUS | Full build, new-business early offset |
| ENDT | Rebuild. Uses the renewal offset if it lands on a pending renewal term |
| FCAN / PCAN | Write-off or waiver of the balance, with no reschedule |
| REIN | Rebuild, plus the $50 reinstatement fee where configured |
| RSTR | Rebuild and expire activities |
| RNEW | Full build, renewal early offset, plan forced to Annual (except P5 and P9) |
| RESC | Used in the PCN rescind path |
| Audits | Adjust the existing schedule |
| PIF | Paid-in-full discount |
| INFO | No impact |

All of these dispatch through one post-policy-term hook. That hook applies the transaction fees: installment setup fee, previously-billed fee, plan switch, write-offs and reinstatement fee.

## Error handling and operations
- A generic async retry batch re-runs failed async requests.
- There are two ways to change a schedule: a full rebuild, or a surgical change (one due date, redistribution, or a single regenerated invoice).
- **Preview projected schedule** is a what-if action that doesn't change anything.

## Company differences
None in the schedule, amounts, thresholds or timing. Only templates, queue names and log prefixes differ.
