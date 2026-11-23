# 0011. One variant that cannot be evaluated makes the whole relation not evaluable

Date: 2026-11-23 · Status: proposed

## Context

ADR 0010 point 3 rules that a relation whose computed range is not finite is "not evaluable". It
yields no verdict and never an *inconsistent*. That rule was written per range. Relations with
variants have one range per variant, so the rule leaves open what happens when one variant's range
is finite and another's is not.

The case that forced the question is independent t with SDs reported to zero decimals:

- `sd = c("0", "0")` gives the rounding interval [0, 0.5] for each SD.
- At SD = 0 the Welch df is 0/0, so the Welch variant of df from (n, SD) is NaN. The Student variant
  is finite: df = n₁ + n₂ − 2.
- If only the Welch row is dropped, df is derived from Student alone (24 for n = 6 and 20). p from
  (t, df) is then evaluated at df = 24 only, while a Welch df could be much smaller. A correctly
  reported Welch result (`t = "0.63"`, `p = ".545"`) then looks impossible.

The result is a false *inconsistent*, the error ADR 0010 ranks worst. The fix was made in step 2 of
ADR 0010 (reprokit 549ee80) without its own decision record; this ADR records it.

## Decision

**If any variant of a relation gives a non-finite range, the relation is not evaluable for every
variant.** In `evaluate_relations()` (reprokit `R/check.R`) this means:

1. All rows already made for that relation are dropped, so no variant of it fits or fails.
2. Its output is not passed on as a derived input to later relations. Otherwise a later relation
   would use a range built from the remaining variants. That range is narrower than the author's
   real choice allows, and the false positive moves one relation further down.

The relation then counts exactly as if an input were missing. This is ADR 0010 point 3 applied to
the relation instead of the single range, for the same reason. Dropping one variant quietly
assumes the author did not choose it, and that assumption is not ours to make. Silence about the
relation is the only answer that is correct whatever the author chose.

Alternatives considered:

| option | why not |
|---|---|
| Drop only the variant that is not finite | Produces the false positive in the context above. |
| Replace NaN by the widest possible range (df in [0, ∞)) | Correct in principle, but needs a domain-specific bound per quantity and relation. NaN carries no ordering to build it from. Can be revisited when a case needs the information. |
| Ignore the relation only when the reported variant is the one that fails | The variant is usually not reported; that is why variants exist. |

## Consequences

- No false *inconsistent* from a variant that is undefined at an edge of the rounding box. The
  regression test is "a variant that cannot be evaluated makes the relation not evaluable, not
  narrower" in `tests/testthat/test-check-t-independent.R`. `inst/validation/check-false-positives.R`
  includes SDs reported to zero decimals and stays at 0 false positives.
- Some checks are lost. With an SD reported as "0", nothing is said about df or about which test
  (Student or Welch) fits, even when the Student part alone would have been informative. This is
  the cost the project accepts: failing to check a correct paper is a limitation, flagging it is a
  false accusation.
- The rule is general. Any future relation with variants (continuity correction, sphericity
  corrections) inherits it without a special case.
