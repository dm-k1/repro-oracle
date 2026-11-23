# 0014. CI confidence levels, paired d conventions, readable variant labels

Date: 2026-11-23 · Status: accepted

## Context

Three gaps in `check_reported()` after ADR 0013.

1. **Only 95% CIs fit.** Papers print 90% and 99% CIs. A correct 90% CI was flagged inconsistent,
   which is a false accusation.
2. **Paired d is not checkable.** It was listed as not checkable, although d_z, d_rm and d_av follow
   from the paired descriptives (Lakens 2013).
3. **Variant codes leak into output.** `two.sided`, `d_av` and the like were printed raw.

## Decision

1. **Level variants.** Every CI relation (one-sample t, paired t, independent t, Pearson) varies
   over the confidence level, `variants = list(level = c("0.90", "0.95", "0.99"))`. A CI fits if a
   level fits; the levels that fit are reported. Only two-sided intervals are checked. `relation()`
   takes variants as a named list by kind (tail, level, test, convention); the old form, a
   character vector plus `variant_kind`, is still accepted and normalised.
2. **Independent-samples CI: crossed variants, one choice per result.** The independent CI depends
   on the test (Student or Welch) and on the level, so each bound is one relation crossed over both,
   `variants = list(test = ..., level = ...)`; `variant_combinations()` evaluates every pair. The
   CI is computed exactly as `t.test()` does (Student SE with Student df, Welch SE with Welch df),
   and the Welch extremum in the SDs is searched per level.
   Crossing alone is not enough: a paper reports t, df and both bounds from one analysis, so a
   Student t with a Welch CI, or a 90% lower bound with a 99% upper bound, must not count as fitting.
   `shared_choice()` in `R/check.R` enforces one choice per result: kinds that are linked by
   falsifying relations are grouped, and an assignment (e.g. test = Welch, level = 0.95) fits only
   if every relation in the group has a fitting row that agrees with it. When no assignment fits,
   the group's relations are inconsistent, even if each would fit on its own choice. The rule holds
   for every handler: both bounds of any CI share one level. Only relations that fit on their own
   take part in the joint choice: a relation that fails under every choice is inconsistent by
   itself, and must not make the numbers of the relations that hold (a correct lower bound, t)
   inconsistent too. This changes which numbers are flagged, never whether a result is.
   *Rejected: split through an intermediate `se` quantity* (se from n and sd per test, CI from m, se
   and df per level). It keeps one kind per relation, but the Student se is no longer paired with
   the Student df, so CI ranges are too wide and fewer planted CI errors are caught (seen in a run
   of `check-planted-errors.R` on that version, which was not kept).
3. **Paired d conventions.** `d_from_descriptives` (m, sd, r: d_z and d_rm) and
   `d_av_from_descriptives` (n, m, sd: d_av and g_av), kind `convention`, `falsifies = FALSE`,
   signed first minus second. A number that no convention reproduces is "not reproducible", never
   "inconsistent". Both relations share the output d.
4. **Shared outputs.** `available[[output]]` is widened (union of ranges) across all relations that
   share an output, so a downstream relation sees the widest value. d verifies if any convention
   of any relation fits.
5. **Implied ranges (semantic change).** `implied_ranges()` (the range a missing input must have)
   now takes the union across the variants of a relation (alternatives) and then the intersection
   across relations, and ignores relations with `falsifies = FALSE`. Before, it intersected across
   variants, which is infeasible once a relation has several levels, and a convention-dependent d
   would have constrained r. The one-choice rule of point 2 is deliberately not applied here: an
   implied range stays the lenient union over choices, so it can only be too wide, never exclude
   a value a correct paper has.
6. **Readable labels.** One map, `variant_labels()` in `R/core-relations.R`, used by
   `handler_choices()` and the check print; `variant_label()` errors on an unknown code, and a
   test asserts every variant of every registered relation has a label.

## Consequences

- Evidence (all in `reprokit/inst/validation/`):
  - `check-false-positives.R`: correct papers at 90/95/99%, Student and Welch, and the paired d
    conventions are never flagged inconsistent (the release gate).
  - `check-planted-errors.R`: detection of planted CI errors per handler, test and level.
  - `check-range-exactness.R`: the crossed CI relations, the paired CI and d relations (d_rm named
    case) give exact ranges.
  - `check-known-answers.R`: sleep data at three levels, paired d_z.
- Detection numbers are recorded in the validation output, not in this ADR.
- Cost of accepting 90/99% CIs: more correct papers fit, so some planted errors become genuinely
  undetectable. With r unknown and three levels to choose from, a paired paper with a shifted M can
  still fit every relation. The independent oracle agrees those cases are undetectable. So the
  hard criterion of `check-planted-errors.R` is now "oracle-detectable and not flagged = 0", and
  the large-error detection rate (formerly a >= 90% criterion) is printed for information only: a
  raw rate cannot tell a regression from an error the reported numbers cannot reveal.
- Cost: crossed relations double the evaluated combinations of the independent CI, each with a
  numeric search for the Welch extremum. Runtime is stated in the script headers.
