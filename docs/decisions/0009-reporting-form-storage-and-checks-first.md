# 0009. Store statistics in reporting form; build check_*() before the remaining handlers

Date: 2026-11-22 · Status: proposed

## Context

The paired-t handler first stored n, M_diff and SD_diff: the minimal sufficient statistics.
Papers almost never report those. They report M and SD per condition (the descriptives
table) and, at best, r between the conditions. Pearson stored only n and r, while papers
report M and SD of both variables next to r.

Two directions use these numbers:

- **share** (`sufficient_stats()`): an author stores statistics for one analysis.
- **check** (`check_*()`): a reader recomputes a result from what a paper reports.

The check direction is the one most people will meet: far more papers are read than
written with reprokit. Its vocabulary (n, M, SD, r, t, F, df, p) is fixed by reporting
practice. If the share side stores a different form, an author cannot fill it from their
own descriptives table, and a reader cannot compare a stored object with the paper.

ADR 0004 does not require minimal storage: a ✅ verdict needs sufficient, round-trip
verified and less identifying than the raw data. M and SD per condition plus r meet all
three.

ADR 0008 kept the build order "all handlers and `appendix()` first, then `check_*()`".
With the check side setting the vocabulary, that order means designing handler storage
before the interface it has to match.

## Decision

1. **Reporting form.** Handlers store statistics in the form a paper reports them, named
   as a paper names them, whenever that form is sufficient:
   - paired t: n, M and SD per condition (named by the condition labels), r;
   - Pearson r: n, M and SD per variable (named by the variable names), r;
   - independent and one-sample t: unchanged (already n, M, SD).
2. **Precision guard (paired t).** SD_diff is rebuilt as
   SD_diff² = SD1² + SD2² − 2·r·SD1·SD2. When r is close to 1 this subtracts two
   near-equal numbers; the relative error is about 1e-16 × (SD1² + SD2²) / SD_diff².
   When that ratio exceeds 1e6, SD_diff is stored as well and used by `reproduce()`.
   Evidence (200 designs per row, scaled difference, ADR 0005):

   | SD_diff / SD | SD_diff stored | worst scaled | result |
   |---|---|---|---|
   | 0.1 | 0/200 | 9.6e-14 | PASS |
   | 0.01 | 0/200 | 1.0e-11 | PASS |
   | 0.003 | 0/200 | 1.26e-10 | PASS |
   | 0.001 | 194/200 | 1.58e-10 | PASS |
   | 1e-4 | 200/200 | 2.26e-11 | PASS |
   | 1e-6 | 200/200 | 1.99e-9 | PASS |

   Without the guard, the round trip failed from SD_diff/SD ≤ 1e-4 (worst 1.43e-7).
3. **Checks first.** Phase F (`check_t()`, `check_F()`, `check_r()` with rounding ranges
   and variants) moves before the rest of Phase C. The remaining handlers (ANOVA, RM
   ANOVA, χ², effect sizes, mediation, SEM) then store what the checks already read.
   This replaces the build-order sentence in ADR 0008; the rest of ADR 0008 stands.

## Consequences

- The coverage table (plan §4) "What is needed" column lists reporting-form statistics;
  paired t and Pearson rows change.
- A stored object for paired t is one number larger than the minimum (M and SD per
  condition instead of M_diff and SD_diff). It identifies no more than the paper's own
  descriptives table, so ADR 0004 condition 3 is unaffected.
- Every new handler states which reported numbers it stores, and its round-trip test runs
  near the numerical edge of its formula (like r → 1 here), not only on typical data.
- Open follow-up: unweighted `lm` stores a covariance matrix. Storing means, SDs and the
  correlation matrix instead would match how papers report predictors, but needs a guard
  for zero-SD columns (intercept, constant dummies). Decide when the checks for
  regression tables are built.
