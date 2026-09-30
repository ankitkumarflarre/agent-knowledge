# Duck Creek Billing: Reference Analyses

These are code-verified analyses of the **current Safepoint Duck Creek Billing** setup (codebase `f:\DuckCreek\Billing`, dated 2026-08-10). The three companies covered are Safepoint (SIC), Cajun (CIC) and Manatee (MIE). The states covered are FL, LA, TX, AL, MS, SC and NC.

> **How to use these:** we are **not** rebuilding Duck Creek 1:1. We keep its business rules and good practices, and we avoid its known defects. The distilled rulebook is
> [../../architecture/billing-rules-from-duckcreek.md](../../architecture/billing-rules-from-duckcreek.md).
> Use the digests below only when you need to trace a rule back to its Duck Creek source.
> Duck Creek names (ManuScripts, `Carrier_SP_BIL_*` procedures, `DC_BIL_*` tables) are for traceability only. Do not copy those names or structures into our code.

| Original | Digest | Covers |
|---|---|---|
| [payplans.html](payplans.html) | [payplans.md](payplans.md) | Pay plans (account and policy plans), eligibility, down payments, auto pay-plan reassignment, endorsement plan switching, fees, and transaction impact |
| [installment-invoice.html](installment-invoice.html) | [installment-invoice.md](installment-invoice.md) | Installment scheduling, due-date rules, amount split, invoice generation and suppression, delivery channels, invoice recovery, and error handling |
| [followup.html](followup.html) | [followup.md](followup.md) | Follow-ups after an invoice: pending-cancellation notice (PCN), notices of cancellation, lapse and withdrawal, collections, reinstatement, equity cancellation, and suppression rules |

Glossary of Duck Creek transaction codes:
- **NBUS**: new business. **ENDT**: endorsement. **RNEW**: renewal. **REIN**: reinstatement. **RSTR**: restart.
- **FCAN**: flat cancellation. **PCAN**: pro-rata cancellation. **RESC**: rescind. **REIS**: reissue. **RWRI**: rewrite.
- **FAUD / RAUD / VAUD / IAUD**: audit transactions. **PIF**: paid in full. **INFO**: information only.

Glossary of Duck Creek notices, activities and charges:
- **PCN**: Pending Cancellation Notice (called "Policyholder Communication Notice" in the source). **RPCN**: renewal PCN.
- **NOC**: Notice of Cancellation. **NOL**: Notice of Lapse. **NOW**: Notice of Withdrawal. **NOCF**: a follow-up PCN after a partial payment.
- **ECN**: a scheduled equity-cancellation activity. **COL**: collections.
- **LTCH**: late charge. **RMDR**: reminder notice. Both are disabled.
- **ISTP**: installment setup fee. **PRBF**: previously-billed fee. **AOI**: additional other interest, such as a mortgagee or premium finance company.
