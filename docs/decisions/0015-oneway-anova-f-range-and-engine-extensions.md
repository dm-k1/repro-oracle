# 0015. One-way ANOVA: exact F range, and the engine changes it needs

Date: 2026-11-24 · Status: accepted

## Context

Build step 5 adds χ² and the between-subjects ANOVAs. The engine was built around t and r, and
one-way ANOVA does not fit it in five places.

1. **Two degrees of freedom.** F has df1 and df2. The vocabulary has one `df`.
2. **No CI in the APA set.** `apa_design()` adds `ci_lower` and `ci_upper` to every design. APA 7
   asks for an F test's effect size, not a CI around F, and χ² has none either.
3. **The number of groups varies.** A t handler has 2 cells (or 1). One-way ANOVA has k ≥ 2, so
   arity cannot be fixed per handler.
4. **Corners are not exact for F, and do not scale.** F from descriptives is
   SSb / SSw · (N − k)/(k − 1), with SSb = Σ nᵢ(Mᵢ − M̄)² and SSw = Σ(nᵢ − 1)SDᵢ².
   - SSw is monotone in each SD, so its corners are exact.
   - SSb is convex in the means. Its maximum lies at a corner of the box of mean intervals, but its
     minimum can lie inside the box: when the mean intervals overlap, SSb can reach 0 and no corner
     does. A corner-only range then has a minimum F above 0, and a correct paper reporting an F
     below it would be flagged inconsistent (the overlapping-means case in
     `check-range-exactness.R` shows a corner-only minimum above 0 where the exact minimum is 0).
   - With k means and k SDs free, 2^(2k) corners pass the 2^14 limit at k = 8.
5. **p from the statistic exists for t only.** `relation_p_from_t()` cannot serve F or χ².

## Decision

1. **`df1` and `df2` are quantities** (domain: positive). The one-way relations give
   df1 = k − 1 and df2 = N − k from the group sizes. The appendix prints F(df1, df2).
2. **The CI is optional in `apa_design()`**: a new argument `ci = TRUE`; ANOVA and χ² handlers pass
   `ci = FALSE`, and then `apa_quantities()` leaves out `ci_lower` and `ci_upper` and the
   appendix prints no CI row. Existing handlers are unchanged.
3. **Variable number of groups.** A handler may declare its per-group quantities without a fixed
   count. The check then requires every per-group quantity that is reported to have the same
   length, at least 2. The t handlers keep their fixed arity.
4. **New quantity `eta2`** (domain [0, 1]). The relation from F, df1 and df2 has kind `convention`
   with variants η² = F·df1 / (F·df1 + df2) and ω² = df1(F − 1) / (df1·F + df2 + 1), and
   `falsifies = FALSE` (ADR 0014 point 3): an effect size that no convention reproduces is "not
   reproducible", never "inconsistent". In one-way designs partial η² equals η², so it is not a
   third variant. Each variant gets a label in `variant_labels()`.
5. **`relation()` takes an optional `range` function** `range(intervals, variant)` that returns
   the exact `c(lower, upper)` of the output over the box. When it is given, `relation_range()`
   uses it instead of corners and stationary points. The one-way F relation uses it:
   - SSw range: SDs at their lower and upper bounds.
   - SSb maximum: the corners of the box of means only (2^k points; SSb is convex, so its maximum
     is at a corner). The sizes nᵢ are integers inside their own intervals; F is evaluated over
     their corners too.
   - SSb minimum: min over a common centre c of Σ nᵢ · dist(c, [loᵢ, hiᵢ])². This is a
     one-dimensional convex piecewise-quadratic function of c, minimised exactly segment by
     segment. It is 0 when the mean intervals share a point.
   - F range = [SSb_min / SSw_max, SSb_max / SSw_min] · df2 / df1.

   `check-range-exactness.R` compares this range with dense random sampling of the box (no sample
   may fall outside it, and the sampled extremes must approach it) and the exact SSb minimum with
   `optimize()`.
6. **One shared p-from-statistic relation.** `relation_p_from_statistic()` replaces
   `relation_p_from_t()`: t (tails two.sided, less, greater, as now), F (upper tail only,
   `pf(F, df1, df2, lower.tail = FALSE)`, no tail choice) and χ² (upper tail, `pchisq`). F and χ²
   are monotone in the statistic, so corners are exact; t keeps its stationary point at 0.
7. **Matching.** The one-way handler matches a formula `y ~ group` with `test = "oneway"`, in the
   same way as the t handlers. `aov` and `lm` objects are matched later.
8. **Wrapper names.** `check_F()` for ANOVA, `check_chisq()` for χ², next to `check_t()` and
   `check_r()` (plan §5.5).

**Factorial ANOVA (decided here, built after χ²).** For a 2×2 design every effect has df1 = 1 and
its SS is a single contrast of the cell means, so F is recomputed from the descriptives exactly,
as above. For effects with df1 > 1 in an unbalanced design the SS depends on the SS type and its
exact range over the box is a quadratic program that base R cannot solve exactly. Those effects
are checked for coherence only (F, df, p, effect size against each other; level 1), with the SS
type as a declared choice.

Alternatives considered:

| option | why not |
|---|---|
| Corners plus stationary points for F | Misses the interior minimum of SSb (false accusation of a correct small F), and 2^(2k) corners stop at k = 8. |
| Intermediate quantities `ssb` and `ssw`, each with its own relation | Each would be exact, but they would appear in the APA and sufficient sets and in the appendix as numbers to report, which they are not. |
| A generic box optimiser (`optim` with bounds) | Not exact: a local optimum gives a range that is too narrow, i.e. a false accusation. |
| Keep `relation_p_from_t()` and add `p_from_F`, `p_from_chisq` | Three copies of one idea; one relation with the statistic as a parameter keeps tails and labels in one place. |
| ω² or partial η² as falsifying | Authors rarely state which they report; a mismatch then accuses a correct paper. |

## Consequences

- Evidence goes in `reprokit/inst/validation/`, not in this ADR:
  - `check-range-exactness.R`: the one-way F range against sampling and `optimize()`.
  - `check-known-answers.R`: PlantGrowth rounded to 2 decimals; the true F and η² lie inside the
    computed ranges.
  - `roundtrip-simulation.R` and `check-designs.R`: one-way designs with k = 2 to 6, balanced and
    unbalanced.
  - `check-false-positives.R` (release gate): correct one-way papers, η² and ω², are never flagged
    inconsistent.
  - `check-planted-errors.R`: planted errors in F, df, p and a group mean.
- The `range` hook is a second way for a relation to be exact. Each relation that uses it needs its
  own entry in `check-range-exactness.R`; the corner-based relations are unaffected.
- With the CI optional, `apa_design()` changes signature; the coverage table and the appendix
  snapshots for the existing handlers must not change.
- χ² reuses decisions 2, 3, 6 and 8; it needs no further engine change unless a continuity
  correction is a variant, which is decided with the χ² handler.
