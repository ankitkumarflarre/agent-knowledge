# Duck Creek Pay Plans: Digest

Source: [payplans.html](payplans.html) ("Billing Pay Plans — Complete Skill & Reference").

## Structure
- There are two levels of plan. An **account plan** (`SP_ACT_A1`–`A10`) groups the **policy plans** (`SP_ACT_P1`–`P25`) that are valid for it. The combination matrix is the only eligibility gate.
- **All three companies share one catalog.** Company differences exist only in presentation and routing: templates, print queues, log prefixes and gateway labels. None of them touch the money math.
- **LOB does not gate pay plans.** LOB only selects the cancellation-notice template. The one exception is the HomeownersAdvantage fee swap described under down payments below.

## Account plan → allowed policy plans

| Account plan | State / model | Policy plans (down payment %) |
|---|---|---|
| A1 | FL base | P1 Annual, P2 Semi-annual 60%, P3 Quarterly 40%, P4 9-Pay 20% (needs an Advantage policy) |
| A2 | FL mortgagee / escrow | P5 Full-pay mortgagee only |
| A3 | LA base | P6 Semi 55%, P7 Quarterly 40%, P8 Annual |
| A4 | Premium finance (LA/TX/FL) | P9 Full-pay premium finance only |
| A5 | TX base | P10 Annual, P11 Semi 60%, P12 Quarterly 40% |
| A6 / A7 / A8 / A9 | LA / AL / TX / MS Advantage | Annual, Semi 55%, Quarterly 31% |
| A10 | SC / NC | P25 Annual only. **There is no installment option.** Intent unconfirmed |

Installment counts: Annual 1, Semi-annual 2, Quarterly 4, 9-Pay 9. All policy plans use commission scheme `POP1` and are eligible for the paid-in-full discount.

## Business rules
- **Eligibility:**
  - A policy plan is valid only if the pair (account plan, policy plan) exists in the matrix.
  - The 9-Pay plan needs `IsAdvantagePolicy`.
  - Custom plans are allowed only when the account has exactly one active term.
  - ⚠ Duck Creek enforces all of this **only by hiding UI options**. There is no server-side check.
- **New business:** choosing a plan cascades the plan settings: max installments, frequency, NB deposit %, commission scheme, PCN allowed, offsets and PIF discount.
- **Renewal:** the renewal is forced to the plan's **Annual sibling**, except for P5 (mortgagee) and P9 (premium finance). Renewal config is nulled for P5 and P9.
- **Endorsement plan switch:** when an endorsement adds or removes a mortgagee or escrow or a premium finance company, the account and policy plan switch. There are about 25 hard-coded branches:

  | State | Condition | Target plan |
  |---|---|---|
  | FL | No mortgagee | A1/P1 |
  | FL | Escrow | A2/P5 |
  | LA | No mortgagee | A3/P8 |
  | LA Advantage | No mortgagee | A6/P13 |
  | TX | No mortgagee | A5/P10 |
  | TX Advantage | No mortgagee | A8/P19 |
  | AL Advantage | No mortgagee | A7/P16 |
  | MS Advantage | No mortgagee | A9/P22 |
  | SC/NC | No mortgagee | A10/P25 |
  | LA/TX/FL | Premium finance added | A4/P9 |

  Removing the mortgagee or premium finance company reverts the plan.
- **Auto pay-plan reassignment after the first payment:** if a partial first payment arrives on an Annual plan, the plan is re-planned to Semi-annual or Quarterly when the payment meets that plan's threshold.

  | Plan group | Semi-annual threshold | Quarterly threshold |
  |---|---|---|
  | P1 | Premium×0.60 + Tax + Fee + 13 | Premium×0.30 + … + 13 |
  | P1 + HomeownersAdvantage | … + MIN(Premium×0.18, 10) | … + MIN(Premium×0.18, 10) |
  | P8 | Premium×0.55 + … + 6 | Premium×0.30 + … + 6 |
  | P10 | Premium×0.60 + … + 18 | Premium×0.30 + … + 18 |
  | P13/P16/P19 | Premium×0.55 + … + band | Premium×0.31 + … + band |

  The fee band is $4 when premium is under $500, $5 for $500–649, and $6 for $650 and above. The same formulas apply to flat-cancelled policies, using the flat-cancellation totals. Mortgagee and premium-finance plans are excluded.
- **Cancellation:**
  - A flat cancellation writes off the open installment setup fee. If an allocation exists, the payment is suspended instead.
  - A pro-rata cancellation holds any overpayment.
  - Balances of $5 or less are written off automatically on pro-rata cancellation.
  - SC/NC charges a **$50 reinstatement fee** after a non-pay cancellation.
- **Reinstatement:**
  - Write-offs of the installment setup fee are reversed.
  - **Automatic reinstatement:** it runs as a service account, with no user action, when all of these hold:
    - the policy was flat-cancelled;
    - a notice of lapse was sent;
    - the due date is 15 days or less away;
    - the suspense balance is at least the invoice total.
- **Fees:**
  - **Installment setup fee** of $10 on the first invoice of non-Annual plans.
  - **Previously-billed fee** on a frequency change: FL $3, LA $6, TX $8.
  - **Invoice threshold:** no invoice is produced below FL/TX $6 or LA/AL/MS/SC/NC $7.
  - Service charges by LOB × form × state, with min/max collars and tiers by outstanding balance.
- **Invoice threshold** is recalculated on every policy-term event.

## Transaction impact

| Transaction | Changes the plan? |
|---|---|
| NBUS | Possibly, through auto-reassignment |
| RNEW | Yes: forced to Annual, except P5 and P9 |
| ENDT | Yes: through the plan-switch matrix |
| FCAN / PCAN | No: write-off or suspense only |
| REIN | No: reverses write-offs |
| Payment and batch payment | Possibly, through auto-reassignment |
| REIS / RWRI | Not implemented |

## Known defects and risks in Duck Creek (avoid)
- **R1:** the endorsement matrix is hard-coded rather than table-driven, and it contains typos.
- **R2:** auto-reassignment fails silently (empty `onFailure`).
- **R6:** a `'SP_ACT,P1'` typo means one branch never matches.
- **R7:** there is no server-side plan validation.
- **R9 / R10:** the company is identified by two separate mechanisms that aren't kept in sync, and the company check is duplicated five or more times.
- **R12:** there is no queryable state/LOB → plan table.
- **R13:** Cajun/Manatee endorsement scenarios were never tested.
- **R14:** SC/NC has no installment option.
- Also missing: the pay-plan path for rewrite and reissue, a refund impact rule, and minimum-premium rules.
