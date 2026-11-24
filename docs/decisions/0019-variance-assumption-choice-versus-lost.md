# 0019. Variance assumption: a declared choice for one-way ANOVA, a `lost` item for factorial

Date: 2026-11-24 · Status: accepted

## Context

- The one-way ANOVA handler (ADR 0015) declares two choices:
  - the effect-size convention;
  - "variance assumption: equal variances (classical F test, not Welch's)".

  ADR 0015 does not mention the second. It was in the handler only, and plan §4 now lists it too
  (commit b91dc92).
- ADR 0017 (factorial ANOVA) declares the SS type and the effect-size convention, and says nothing
  about variances.
- The question is whether the factorial row needs the variance choice too.

A choice to declare tells the reader which of the plausible analyses was run, when they give
different numbers:
- **One-way:** Welch's ANOVA (`oneway.test()`) is a common alternative. It changes F and df2.
- **Factorial:** there is no mainstream Welch-type alternative. Heteroscedasticity-adjusted F (for
  example `car::Anova(white.adjust = TRUE)`) is rare in papers.

## Decision

**In one line:** equal variances is a declared choice for one-way ANOVA and an unchecked assumption
for factorial ANOVA.

1. **One-way ANOVA keeps the variance assumption as a declared choice**, in the handler and in
   plan §4.
2. **Factorial ANOVA does not declare it.** Its choices stay the SS type (unbalanced designs only)
   and the effect-size convention, as ADR 0017 has them.
3. **The factorial handler's `lost` list states "equal variances assumed (pooled error term)".** It
   is an assumption of the analysis that the check does not test. This is how ADR 0016 treats χ²'s
   expected-count assumption.

Alternatives considered:

| option | why not |
|---|---|
| Declare the variance assumption for factorial too, to match one-way | Asks the author to choose between options almost no paper uses. The choices column is for analyses that plausibly differ. |
| Say nothing about variances for factorial | The pooled error term is an assumption that the shared summaries cannot test, so it belongs in `lost`. |

## Consequences

- When the factorial handler is built (ADR 0017), its `lost` list carries the item from point 3.
  It needs a test that checks this.
- Plan §4's factorial row is unchanged.
- If a heteroscedasticity-robust factorial test becomes common in the papers checked, this is
  revisited as a `test` variant.
