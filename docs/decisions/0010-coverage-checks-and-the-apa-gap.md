# 0010. Checks report coverage; the appendix reports the gap beyond APA

Date: 2026-11-23 · Status: proposed

## Context

Plan §6 described the check direction the way statcheck works: recompute p from the reported
statistic and df. That makes p the only number under test. Every other reported number (M, SD, n,
CI, effect size) is an input or ignored. A reader with a full descriptives table learns no more than
a reader with only "t(29) = 2.85, p = .008".

Two things follow from how papers actually report results.

1. **What can be checked depends on which numbers are present.** A paper that reports M, SD and n
   per condition allows the statistic itself to be recomputed, not only p. A paper that reports only
   the statistic allows internal coherence only. The check must say which of the two it did, per
   reported number, instead of returning one verdict.
2. **APA is the baseline, and for some tests it is not enough.** APA 7 / JARS fixes which numbers a
   paper reports. For a one-sample t or a Pearson r that set is self-contained: the result is
   reproducible from it. For a paired t it is not — APA does not ask for r between conditions, the
   one number that makes t recomputable. For RM ANOVA, regression and SEM the gap is much larger (a
   covariance matrix). The size of that gap is the project's argument: the more complex the design,
   the more a paper must add beyond APA, until the summaries approach the raw data.

The share and check directions need the same vocabulary for this (ADR 0009). What was missing is the
structure that links the numbers to each other.

## Decision

1. **Relation graph per handler.** Each handler declares, in the registry, three new fields over one
   shared vocabulary of quantity names (`n`, `m`, `sd`, `r`, `t`, `f`, `df`, `p`, `ci`, `d`):
   - `apa`: the quantities APA 7 / JARS requires for this test;
   - `reproducible`: the quantities that determine every reported result (the sufficient set, the
     same numbers `summarise` stores);
   - `relations`: the equations between quantities, each with inputs, one output and a function.
2. **Checks report coverage, not one verdict.** `check_reported()` evaluates every relation whose
   inputs are present and returns:
   - a status per reported number — *verified* (recomputed from more basic reported numbers),
     *coherent* (fits the other derived numbers), *inconsistent*, or *not checkable*;
   - a coverage level: 0 nothing checkable, 1 internal coherence of statistic/df/p/CI,
     2 the statistic recomputed from the descriptives, 3 the reported set contains `reproducible`;
   - which missing numbers would raise the level.
   `check_t()`, `check_F()` and `check_r()` stay as thin wrappers (plan §5.5).
3. **Missing quantities are solved for, not skipped.** When a relation's output is reported and one
   input is missing, the check reports the range of that input consistent with the rest
   ("t is consistent only if r between conditions lies in [.41, .58]"). It is inconsistent only when
   no admissible value works. All intervals come from the reported decimals (rounding ranges, plan
   §6), not from a fixed tolerance.
4. **The appendix is APA plus the gap.** `appendix()` prints the APA set, then
   `reproducible` minus `apa` with a one-line reason per added number. The same two fields generate a
   **reporting-gap table** for the README, next to the coverage table.
5. **Pearson keeps M and SD, as `apa` only.** r is not recomputable from means and SDs, so M and SD
   are not part of `reproducible` for Pearson: n and r are. They are stored because the APA
   descriptives table reports them and the check side reads them. The plan §4 row is split into what
   is needed and what is reported anyway. This refines ADR 0009 item 1; the rest of ADR 0009 stands.

## Consequences

- `register_handler()` gains validation: `reproducible` must be a subset of the names `summarise`
  stores, so a handler cannot claim reproducibility from numbers it does not keep.
- Every new handler is built with its relations, so the check side and the share side can no longer
  drift apart. This is also why `check_*()` comes before the remaining handlers (ADR 0009 item 3).
- The check direction needs its own validation, separate from the round trip: known-answer cases
  rounded from real `t.test()` / `cor.test()` output must never be flagged, planted errors must be,
  and the detection rate is reported per relation (plan §8).
- Interval evaluation over the corners of the rounding box is exact only for relations monotone in
  each input. Each relation is tested for that; the non-monotone ones (solving for r) are optimized
  over the box instead.
- "What is needed" in the coverage table and "what APA gives" are now two different columns. Rows
  where they coincide (one-sample t, Pearson) are the honest ✅; rows where they do not (paired t,
  RM ANOVA, lm, SEM) carry the project's message.
