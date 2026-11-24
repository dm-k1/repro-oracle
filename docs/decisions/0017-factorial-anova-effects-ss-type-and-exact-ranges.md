# 0017. Factorial ANOVA: one effect per check, SS type, and which F ranges are exact

Date: 2026-11-24 · Status: accepted

## Context

Build step 5 ends with the factorial between-subjects ANOVA. ADR 0015 decided its outline: a 2 × 2
design is recomputed exactly, and effects with df1 > 1 in an unbalanced design are checked for
coherence only, with the SS type as a declared choice. Five things are still open, and one part of
ADR 0015's outline is too broad.

1. **Several results per test.** A two-way ANOVA reports three F tests (A, B, A × B), each with
   its own F, df1, p and effect size, but one set of cell descriptives and one df2. Every handler so
   far has one statistic.
2. **The cells have a shape.** The cell means, SDs and n are an a × b table. A flat vector of 6
   cells can be 2 × 3 or 3 × 2, and the shape decides which cells form which effect.
3. **The SS type.** In an unbalanced design, types I, II and III give different SS for the main
   effects. ADR 0015 made it a declared choice, but not whether the check tries every type.
4. **The effect size.** In a one-way design the formula from F gives η², which equals partial η².
   In a factorial design the same formula gives partial η², and plain η² (SS_effect / SS_total)
   needs the SS of the whole design.
5. **Which ranges are exact.** ADR 0015 implies that every effect of a balanced design is exact.
   That holds for effects with df1 = 1 and for main effects, but not for an interaction with
   df1 > 1 (point 4 of the Decision).

## Decision

**In one line:** one check per effect, with the effect named as a test argument. The cells are an
a × b matrix, and its `dims` decide the shape. F is exact when the effect is one contrast (df1 = 1)
or a balanced main effect; otherwise the check is coherence only. The SS type is a falsifying
variant. The effect size is partial η² or partial ω², a convention that does not falsify.

1. **Two factors in v1.** The handler `anova_factorial` matches `y ~ a * b` with
   `test = "factorial"`. Three or more factors, and within-subject factors, come later (RM ANOVA is
   triage-only in v1, ADR 0013 point 7).
2. **One effect per check: test argument `effect`**, one of `"A"`, `"B"` or `"AB"`. In the print
   and the appendix these are shown by the factor names.
   - The quantities stay scalar (`f`, `df1`, `df2`, `p`, `eta2`), so the vocabulary and the
     relations of the one-way handler are reused.
   - A paper with three F tests gets three calls of `check_F()`. Each call gets the same cell
     descriptives.
   - `appendix()` on the share side prints all three effects, because it has the data.
   - `effect` is declared in the handler's test arguments (ADR 0016 point 4). Without it, the call
     is an input error. Nothing is guessed.
3. **The cells are a matrix, as for χ².**
   - `m`, `sd` and `n` are passed as a × b matrices. The wrapper reads `dim()`, checks that the
     reported matrices agree, and passes `dims = c(a, b)` on (ADR 0016 point 3).
   - Rows are the levels of A and columns the levels of B. The flat order is R's column-major order.
   - df1 is a − 1, b − 1 or (a − 1)(b − 1). df2 is N − ab.
   - Without `dims`, df1 is not checkable, and F is checked for coherence with df1, df2 and p only.
4. **Which F ranges are exact.** The SS of an effect is a function of the cell means; SSw is
   Σ(nᵢⱼ − 1)SDᵢⱼ² as in one-way, and is monotone in each SD.
   - **df1 = 1 (every effect in a 2 × 2 design, and the main effect of a two-level factor in any
     design):** SS_effect = (L·μ)² / (L·D⁻¹·Lᵀ), with L one row of contrast weights over the cells
     and D the diagonal of cell sizes. L·μ is linear in the means, so its range over the box is
     exact by interval arithmetic. The square's minimum is 0 when that range contains 0 and
     otherwise lies at the nearer end. This holds in balanced and unbalanced designs and for every
     SS type: only L changes (point 5).
   - **Balanced main effect with df1 > 1:** SS_A = n·b·Σ(Āᵢ − Ā)². The marginal means Āᵢ are
     averages of disjoint cells, so their intervals are exact and independent. SS_A is then the
     one-way SSb of the marginal means, and ADR 0015's exact range hook applies unchanged.
   - **Interaction with df1 > 1, and any effect with df1 > 1 in an unbalanced design:** SS_effect
     is a convex quadratic form in the means. Its maximum lies at a corner of the box, but its
     minimum is a quadratic program with a null space larger than one dimension, so ADR 0015's 1-D
     method does not apply. These effects are checked for coherence only (F, df1, df2, p and
     partial η² against each other; level 1).
   - **The correction to ADR 0015:** a balanced interaction with df1 > 1 (e.g. 2 × 3) is
     coherence only too, not exact.
5. **SS type: variant kind `test`, values `III`, `II` and `I`, falsifying.**
   - In a balanced design all three types agree, so there is one value and no variant.
   - For the interaction of a two-factor design, all three types also agree (it is the last term
     in every type).
   - For a main effect in an unbalanced design, the types give three different L:
     - III: unweighted marginal means.
     - II: A adjusted for B.
     - I: A entered first, not adjusted. Type I with A entered second equals type II, so it is not a
       fourth value.
   - It falsifies for the same reason as Yates (ADR 0016 point 5): each type is an exact function
     of the descriptives, so an F that fits none is an inconsistency.
   - F, p and partial η² share one choice through `shared_choice()`.
   - The appendix asks the author for the type. The variant shows which types fit.
6. **Effect size: `eta2`, kind `convention`, `falsifies = FALSE`, with the variants partial η² and
   partial ω².**
   - partial η² = F·df1 / (F·df1 + df2). This is the one-way η² formula from ADR 0015 point 4. For
     an all-between design it also equals generalised η².
   - partial ω² = df1(F − 1) / (df1(F − 1) + N). This is ADR 0015's ω², because df1 + df2 + 1 = N.
   - Each variant gets a label in `variant_labels()`. The one-way handler keeps the labels η² and
     ω², because there they equal the partial forms.
   - Plain η² is not a variant in v1. It needs SS_total from the cell descriptives, and its range is
     a ratio of two functions of the same means, which corners do not make exact. A paper that
     reports plain η² therefore gets "not reproducible", never "inconsistent".
7. **Sets (plan §4).** The APA set has the cell M, SD and n, plus F, df1, df2, p and the effect size
   per effect. The sufficient set is the cell M, SD and n, so nothing is missing (ADR 0013,
   Consequences). The choice to declare is the SS type, and it only matters when the design is
   unbalanced.
8. **`reported()`** uses base R only:
   - type I from `anova(lm())` in both factor orders;
   - type II from `drop1()` on the additive model for the main effects;
   - type III from `drop1()` with `contr.sum` contrasts.

   No package for this goes into Imports.

Alternatives considered:

| option | why not |
|---|---|
| Quantities per effect (`f_A`, `f_B`, `f_AB`) | Triples the vocabulary and its "missing" reasons, and does not extend to three factors. |
| One call with vectors of F, p and η² per effect | Needs vector-valued statistics in the engine for one handler. Three calls give the same verdicts. |
| Exact range for every effect with a QP solver | Base R has no exact QP. A local optimum gives a range that is too narrow, i.e. a false accusation (ADR 0015). |
| The outer range [0, corner max] for the non-exact effects | It never accuses wrongly and can catch an F that is too large. But it needs a third status between "verified" and "coherent". It can be added later without changing this ADR. |
| SS type as a convention (`falsifies = FALSE`) | Each type is exact. Not falsifying would let any wrong F in an unbalanced design pass as "not reproducible". |
| Plain η² as a variant with an interval-quotient range | A sound but wide range. It would "reproduce" wrong values and give no detection for a convention that cannot falsify anyway. It comes later if papers need it. |
| Infer the shape from df1 and the number of cells | Same ambiguity as for χ² (ADR 0016). |

## Consequences

- **Evidence goes in `reprokit/inst/validation/`, not in this ADR:**
  - `check-known-answers.R`:
    - `ToothGrowth` (supp 2 × dose 3, balanced): supp is exact (df1 = 1), dose is exact (balanced
      main effect), supp × dose is coherence only.
    - `mtcars` am × vs for mpg (2 × 2, unbalanced): types I, II and III each fall inside their
      computed range, and each is compared with `anova()` and `drop1()`.
  - `check-range-exactness.R`: two new entries. They compare the df1 = 1 contrast range and the
    balanced marginal-means range with dense sampling of the box.
  - `roundtrip-simulation.R` and `check-designs.R`: 2 × 2 to 3 × 4, balanced and unbalanced, each
    SS type.
  - `check-false-positives.R` (release gate): correct papers with each SS type and with partial η²
    or partial ω² are never flagged inconsistent.
  - `check-planted-errors.R`: a changed cell mean, a digit in F or p, df2 from the wrong N, and a
    type III F paired with a type II p.
- **Engine:**
  - test argument `effect`;
  - an `ss_type` variant kind with its labels;
  - labels for partial η² and partial ω²;
  - no new quantity.
- **`check_F()`** gets the factorial form. Its roxygen shows one 2 × 2 example with all three calls.
- **ADR 0015's factorial paragraph is narrowed by point 4.** ADR 0015 is otherwise unchanged. The
  plan's factorial row is checked against point 7 when the handler is built.
- **The coverage table shows one row** for the factorial ANOVA, not one per effect, because the
  sets are the same for each effect.
