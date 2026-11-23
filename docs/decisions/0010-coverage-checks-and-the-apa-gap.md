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
2. **APA states the obligation and declines to make it concrete.** Checked against the Publication
   Manual (7th ed.), §3.7 and §6.43. APA does not give per-test reporting sets. It gives one
   obligation plus one generic list, and says explicitly that the general case cannot be specified:

   > "it is impossible to specify what constitutes a 'sufficient set of statistics' in general
   > terms. However, such a set usually includes at least the following: per-cell sample sizes,
   > observed cell means (or frequencies of cases in each category for a categorical variable), and
   > cell standard deviations or pooled within-cell variance." (§3.7)

   > "The data supplied […] should allow readers to confirm the basic reported analyses (e.g., cell
   > means, standard deviations, sample sizes, correlations) […] What constitutes sufficient
   > information depends on the analytic approach reported." (§6.43)

   So the project is not working against APA or beyond it. It is doing the thing APA says is
   impossible in general terms and necessary in each particular case: fixing, per analysis, what a
   sufficient set is, and checking whether a paper's numbers meet it. APA already owns the
   vocabulary (`sufficient_stats()` is APA's own phrase), already demands that readers be able to
   "confirm the basic reported analyses" (the check direction), and already names the deliverable:

   > "At times, the amount of information that constitutes a sufficient set of statistics is
   > extensive; when this is the case, the information could be supplied in a supplementary data set
   > or an appendix." (§3.7)

   The gap is therefore between APA's *generic* list and what each test actually needs. Paired t is
   the sharpest case: APA's "usually includes at least" list is per-cell n, M and SD, and from those
   three a paired t cannot be recomputed — the correlation between conditions is missing, and the
   manual addresses it nowhere (no occurrence of "correlation between"; Table 3.1 and §6.43 omit
   it). For RM ANOVA, regression and SEM the gap is larger still, and §3.7 is explicit that there a
   variance-covariance or correlation matrix is what suffices.

The share and check directions need the same vocabulary for this (ADR 0009). What was missing is the
structure that links the numbers to each other.

## Decision

1. **Relation graph per handler.** Each handler declares, in the registry, three new fields over one
   shared vocabulary of quantity names (`n`, `m`, `sd`, `r`, `t`, `f`, `chisq`, `df`, `p`,
   `ci_lower`, `ci_upper`, `d`):
   - `apa`: APA's generic baseline applied to this test — the test statistic, its df, the exact p,
     the effect size with its CI, and per-cell n and M with SD (or frequencies, for a categorical
     variable). It is derived from one rule for every handler, not curated per test, because APA
     has no per-test sets to curate (see Context 2). A handler may not add to it on a hunch: an
     item belongs in `apa` only if §3.7, §6.43 or JARS Table 3.1 puts it there.
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
4. **The appendix is APA plus the gap.** It opens by citing §3.7 and §6.43, so an author can see the
   requirement it discharges rather than take the package's word for it. `appendix()` prints the APA set, then
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
- Interval evaluation over the corners of the rounding box alone is **not** exact: two v1 relations
  are non-monotone inside realistic boxes (t from descriptives, where ∂sd_diff²/∂SD₁ vanishes at
  SD₁ = r·SD₂; and two-sided p from t, whose maximum sits at t = 0 when the interval straddles it).
  A relation therefore also declares its interior stationary points, and the range is the extreme
  over corners *and* those points — exact for smooth functions. The claim is verified by sampling
  the box densely and asserting no sampled value falls outside the computed range, not by assuming
  monotonicity. Solving for a missing input is a grid scan over its admissible range, because the
  answer wanted is the feasible set, not an extremum.
- "What is needed" in the coverage table and "what APA gives" are now two different columns. Rows
  where they coincide (one-sample t, Pearson) are the honest ✅; rows where they do not (paired t,
  RM ANOVA, lm, SEM) carry the project's message.

## Sources

Publication Manual of the American Psychological Association, 7th ed., read directly (the copy
consulted is reflowed and carries no printed page numbers, so sections are cited, not pages):

- **§3.7, Reporting Results of Inferential Statistical Tests** — the "sufficient set of statistics"
  passage quoted in Context 2; the generic per-cell list; variance-covariance or correlation
  matrices as the sufficient set for multivariable systems; "the information could be supplied in a
  supplementary data set or an appendix"; "enough statistical information to allow its inclusion in
  future meta-analyses"; statistic + df + exact p + size and direction of the effect; a point
  estimate always with a named measure of variability.
- **§6.43, Statistics in Text** — data supplied "should allow readers to confirm the basic reported
  analyses (e.g., cell means, standard deviations, sample sizes, correlations)"; "What constitutes
  sufficient information depends on the analytic approach reported."
- **§6.44** — confidence-interval format, and that the level of confidence must be stated.
- **JARS–Quant Table 3.1** (the only JARS table printed in the manual; others are on the APA
  website) — exact p values; "the minimally sufficient set of statistics […] needed to construct the
  tests"; effect sizes and CIs "when possible". It has no per-test rows.
- **§1.14, Data Sharing** — "Authors must make their data available after publication", and the
  Ethics Code (Standard 8.14) prohibits withholding data from qualified requesters for verification
  through reanalysis. This is the basis for the project's fixed line that sharing the raw data
  remains the standard: it is APA policy, not this project's opinion.

What the manual does **not** address, checked by searching the text: the correlation (or covariance,
or SD of differences) for a paired t; Student versus Welch ("Welch" does not occur); sphericity,
epsilon or corrected df for RM ANOVA ("sphericity" does not occur); which d variant or Hedges' g,
and whether the effect-size convention must be declared ("Hedges" does not occur); an effect size
for chi-square. Two consequences:

1. The project's rule that an effect size is reported only with a declared convention goes beyond
   APA deliberately. It is our choice, and the appendix must say so rather than imply APA requires
   it.
2. For these items the evidence is absence, not prohibition. The honest wording in outputs is "APA
   does not require this", never "APA forbids" or "contrary to APA".

Unverified: the sample statistical tables (Tables 7.8–7.10) are images in the copy consulted and
were not read. They may name columns relevant to the regression and ANOVA rows of the gap table.
