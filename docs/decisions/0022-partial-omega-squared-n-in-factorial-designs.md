# 0022. Partial ω² in factorial designs: N is df2 + number of cells

Date: 2026-11-24 · Status: accepted

## Context

- ADR 0017 point 6 gives partial ω² = df1(F − 1) / (df1(F − 1) + N) and takes N from ADR 0015
  "because df1 + df2 + 1 = N".
- That identity holds only for a one-way design, where df1 = k − 1 and df2 = N − k.
- In an a × b design the error df is df2 = N − ab, while df1 is the df of one effect. So
  df1 + df2 + 1 ≠ N. For a 2 × 3 design and effect A, df1 + df2 + 1 = N − 4.
- Using df1 + df2 + 1 would make the partial ω² variant fail to reproduce correctly reported values.
  It is a convention that does not falsify, so the result would be "not reproducible" instead of a
  false accusation, but the variant would still be wrong.
- The factorial implementation (build step 5b, reprokit commit 0ace1c7) found this.

## Decision

**In one line:** in the factorial handler, partial ω² uses N = df2 + a·b.

1. **The formula is partial ω² = df1(F − 1) / (df1(F − 1) + df2 + a·b)**, with a·b the number of
   cells from `dims`.
   - Without `dims`, N is unknown, and the partial ω² variant is not evaluable.
   - The one-way handler is unchanged: there, a·b is k, and df2 + k = df1 + df2 + 1.
2. **The `eta2` domain is [−1, 1] in the factorial handler**, through a per-handler domain (ADR
   0016 point 1). Partial ω² is negative when F < 1, and a correct paper can report a negative
   value.

## Consequences

- ADR 0017 point 6 is amended on N only. Its other points stand.
- The known-answer and false-positive scripts in `reprokit/inst/validation/` cover partial ω² with
  the corrected N (commit eafe4cb).
- **Resolved (2026-11-25):** the one-way handler had the same problem. A correctly reported
  negative ω² (for example F = 0.00, ω² = −.07) made the check stop with an input error, so a
  correct paper could not be checked at all. The one-way handler now also has the `eta2` domain
  [−1, 1] (reprokit commit 186e2c3), and `check-false-positives.R` includes one-way designs whose
  ω² prints negative.
