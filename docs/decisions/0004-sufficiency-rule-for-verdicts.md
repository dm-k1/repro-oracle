# 0004. One rule decides each analysis's verdict

Date: 2026-11-22 · Status: accepted (changes plan §3, §4, §12)

## Context
The coverage table (§4) gave verdicts row by row, without a stated rule. Two problems came
up:

- Some rows were marked ⚠️ although the summaries they need are close to the raw data
  (per-person sums for growth models, per-cluster summaries for small clusters). Offering
  those as an alternative to sharing gives readers less information without protecting
  anyone, which is the excuse the tool must not become (§1).
- Some rows mixed reproduction with replacement: a Monte Carlo CI for mediation is a
  different method from the bootstrap CI it would "reproduce".

## Decision
An analysis gets **✅ summaries suffice** only if all three conditions hold:

1. **Sufficient**: the summaries mathematically determine the reported results.
2. **Verified**: a round-trip test (data → kit → refit) matches the original to ~1e-8.
3. **Less identifying**: the summaries are clearly less identifying than the raw data.

If one condition fails, the analysis moves down:

| Verdict | Condition | Told to the author |
|---|---|---|
| ✅ Summaries suffice | 1, 2 and 3 hold | report these statistics; readers can reproduce the result exactly |
| 🔁 Synthetic data | summaries are not sufficient, but a synthetic dataset can approximate the result (quality measured by refitting) | approximate only; share the raw data if you can |
| 🔒 Needs the raw data | neither works, **or** the needed summaries are close to the raw data | summaries give no real privacy gain here, only less information; share the raw data |

Further rules:

- **No round-trip test, no claim.** An analysis that has not been validated cannot be ✅,
  even if the math works on paper (random intercept in v1, random slopes).
- **A replacement method is labelled as a replacement**, never as reproduction (Monte Carlo
  CI for a bootstrap CI).
- **Effect sizes are ✅ only with a declared convention.**
- **Borderline case**: χ² and logistic/Poisson with categorical predictors stay ✅, but carry a
  fixed note "for these variables the table *is* the data", and the appendix must show the
  disclosure check.
- Every verdict ends with "sharing the raw data remains the standard".

## Consequences
- + Every row in the coverage table can be defended in one sentence by pointing at the
  condition it fails.
- + Condition 2 is already enforced in code: `register_handler()` refuses a
  `summaries_suffice` handler without kit, refit and example.
- + The class pattern gets sharper: beyond simple between-group tests and linear models,
  analyses drop to 🔁 or 🔒.
- − Condition 3 is a judgement, not a formula. Where it depends on the data (cluster size,
  cell counts), the handler's `triage()` must decide it per dataset and say why.
- − Rows moved: mixed/growth with missing time points ⚠️ → 🔒; bootstrap CI ⚠️ → 🔒 (with
  Monte Carlo as replacement); linear random intercept ⚠️ → 🔁 in v1. SEM split by estimator.
  Nonparametric tests added as 🔒.
