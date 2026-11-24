# 0020. Regression family: handlers per plan row, multi-term designs, check relations

Date: 2026-11-24 (revised and accepted 2026-11-25) · Status: accepted

## Context

Build step 6 adds simple regression, multiple regression, moderation and simple mediation (plan §4,
§13). ADR 0018 settled their APA sets. Each step 5 handler got a design ADR before it was built
(0015, 0016, 0017), and step 6 needs one too, because the engine does not fit regression yet:

1. **There is one coverage row per handler, but four plan rows.** `coverage_table()` builds one row
   per registered handler. The existing `lm` handler covers simple, multiple and moderation in a
   single entry: OLS and WLS, numeric and factor predictors, weights and offset (ADR 0007). Their
   APA sets, missing quantities and choices differ, so one entry cannot give the three rows.
2. **`lm` has no `design`.** So it has no APA set, no coverage row and no check route. It only
   shares and reproduces.
3. **`apa_design(statistic, effect_size, cells, ci)` assumes one test.** A regression reports b, SE,
   t, p and often a CI for every coefficient, plus a model-level R² and F. ANOVA solved a variable
   number of cells with `cells = NA`. Regression has a variable number of terms *and* a second
   level.
4. **Mediation has no model object.** It is three regressions: M on X, Y on X, and Y on X and M.
   Plan §5.3 says tests whose result objects do not hold the data come in as formula + data.
5. **Which reported numbers determine which.** All of these are exact for OLS:
   - t = b / SE;
   - p from t and df_resid;
   - CI = b ± t_crit · SE;
   - F = (R² / df1) / ((1 − R²) / df2), with df1 = k and df2 = n − k − 1;
   - simple regression only: R² = r², t² = F, and r = b · SD_x / SD_y;
   - mediation: each path a, b, c and c′ is a regression coefficient with its own SE, t and p.
6. **The indirect effect is inferred by a method the summaries cannot reproduce.**
   - Papers test a·b with a bootstrap CI, which resamples cases (plan §4: "summaries cannot").
   - Sobel-type z tests can be computed from a, b and their SEs, but the field has replaced them
     with the bootstrap. Checking them would endorse a test that the paper usually did not run.

## Decision

**In one line:** each plan row gets its own handler, all sharing the lm code. `apa_design()` gains
a list of terms and a model level. Checks are coherence among the reported numbers. For mediation,
only the in-model paths a, b, c and c′ are checked, and nothing is said about the indirect effect.
Adjusted R², β and centering do not falsify.

1. **There are four handlers, one per plan row.**
   - `lm_simple`, `lm_multiple` and `lm_moderation` are registered with their own `matches`, APA
     set, sufficient set, missing reasons and choices.
   - All three reuse `lm_summarise()`, `lm_reproduce()` and `lm_reported()`. There is no second
     implementation of lm.
   - What each matches, on the terms of an unweighted or weighted `lm`:
     - **simple:** one numeric predictor;
     - **moderation:** exactly one two-way product term, with both of its main effects, where each
       factor in it has two levels;
     - **multiple:** two or more predictor columns and no product terms. Factors are allowed.
   - Every other lm stays with the general `lm` handler. It keeps triage, share and reproduce, but
     has no coverage row and no check route in v1.
   - Registry order puts the narrow handlers before `lm`, so the most specific match wins.
2. **Mediation is the handler `mediation_simple`, through the formula route.**
   - Share: `y ~ x` with `test = "mediation"` and the test argument `mediator = "m"` (ADR 0016
     point 4), plus `data`.
   - Stored: n, the means and the covariance matrix of X, M and Y.
   - Reproduced: the three regressions (M on X; Y on X; Y on X and M), with b, SE, t and p for the
     paths a, c, b and c′. No indirect effect, no Sobel test.
   - Two or three fitted lm objects as input come later.
   - The indirect effect a·b and its CI are "not checkable" ("inferred by bootstrap, which
     resamples cases; summaries cannot"). The bootstrap CI goes in `lost`.
3. **`apa_design()` gains `terms` and `model`.**
   - `terms = NA_integer_` means a variable number of coefficients, like `cells = NA` in ADR 0015.
     The per-term quantities are b, SE, t, p and, when `ci = TRUE`, the CI.
   - `model = TRUE` adds the model-level quantities: R², F, df1, df2 and the model p.
   - The effect size is R² at the model level. β per term is a convention (point 6).
   - The existing designs are unchanged, because both new arguments default to "absent".
4. **Check relations for the three lm handlers, from reported numbers only (level 1 coherence):**
   - t from b and SE. This is an interval quotient, and it is monotone in each argument, so its
     corners are exact.
   - p from t and df_resid, two-sided (`relation_p_from_statistic("t")`).
   - The CI from b, SE, df_resid and the level. The levels are those of ADR 0014.
   - F from R², df1 and df2, and p from F (`relation_p_from_statistic("f")`).
   - n implied by df_resid = n − k − 1 (ADR 0012). k is a declared test argument, read from df1
     when F is reported. Without k, n is not checkable. It is never guessed from the number of
     terms given, because terms can be left unreported.
   - **simple only:** R² = r², t² = F, and r = b · SD_x / SD_y. The last one uses the M and SD
     from the APA set (ADR 0018).
5. **The check wrapper is `check_lm()`.**
   - Per-term quantities are vectors named by term: `b = c(wt = "-3.19", cyl = "-1.51")`, and SE,
     t and p alike.
   - The `handler` argument chooses the row, defaulting to `lm_multiple`. `check_mediation()`
     covers mediation.
6. **Conventions that do not falsify** (`falsifies = FALSE`):
   - **adjusted R²:** a variant of the model effect size, R² or adjusted R². It is an exact
     function of R², n and k. Papers often label it plain R², so a mismatch is "not
     reproducible", not "inconsistent".
   - **β (standardised coefficient):** β = b · SD_x / SD_y needs the SDs from the APA set. Papers
     use "b" and "β" inconsistently, so a reported β that does not fit means "not evaluable as
     labelled". This is the ambiguity-towards-silence rule.
7. **Mediation checks only the in-model paths.**
   - Each path (a, b, c, c′) is checked as a regression coefficient with the point 4 relations: t
     from b and SE, p from t and df_resid, and the CI. df_resid is n − 2 for a and c, and n − 3 for
     b and c′.
   - No relation involves a·b. The identity c = c′ + a·b is not used, because it is a statement
     about the indirect effect.
   - Sobel, Aroian and Goodman z tests are not computed, not checked and not offered as variants.
   - A reported a·b or indirect-effect CI is listed as not checkable. The print says why, and never
     marks it as fitting or not fitting.
8. **Choices to declare** (unchanged from plan §4):
   - multiple regression: the factor coding;
   - moderation: centering and coding;
   - mediation: none in v1. The indirect effect's CI method would be the choice, but the indirect
     effect is not checked (point 7).

   None of them can falsify from reported numbers. Centering changes the lower-order b but not the
   interaction b, and the check uses no relation that depends on it. The choices go to the
   appendix and to reproduce.
9. **Each handler carries the source comment of ADR 0018 point 2**, saying which part of `apa` is
   JARS and which is §3.7.
10. **Not in v1:**
    - standardised coefficients as a falsifying check;
    - ΔR² for hierarchical steps;
    - partial and semi-partial correlations;
    - heteroscedasticity-robust SEs (they go in `lost`);
    - any check of the indirect effect (a·b, Sobel-type tests, bootstrap or Monte Carlo CIs);
    - multiple mediators;
    - moderated mediation;
    - Johnson–Neyman intervals;
    - simple slopes as a check. Simple slopes need the covariance of the coefficients, which papers
      do not report.

Alternatives considered:

| option | why not |
|---|---|
| One `lm` handler that emits several coverage rows | Breaks "one row per handler" in the registry and in `coverage_table()`, and the per-row `matches` logic would live inside one handler anyway. |
| Make the general `lm` the multiple-regression row | Then every lm (polynomials, three-way interactions, weighted fits with an offset) would claim the v1 multiple-regression APA set, which does not describe them. |
| A separate `mediation()` share function | Plan §5.3 already has the formula route for tests without a model object, and ADR 0016 gave it test arguments. A new entry point duplicates it. |
| Sobel family as a falsifying variant | It checks a test the paper usually did not run, because the bootstrap replaced it. A matching Sobel z would lend credibility to an inference the summaries cannot reproduce. |
| Check a·b as a product of a and b, or c = c′ + a·b | Both are claims about the indirect effect. Staying silent on it keeps the message simple: the paths are checkable, the indirect inference is not. |
| β as falsifying | Correct papers that write "β" for unstandardised b would be flagged. That breaks the false-positive gate. |
| Infer k from the number of reported terms | Papers often omit control variables from the table, so this accuses correct papers. |

## Consequences

- **Engine changes:**
  - `apa_design()` gets `terms` and `model`;
  - per-term quantities are named by term;
  - new quantities b, se, r2, r2_adj (variant label) and beta. The mediation paths are terms named
    a, b, c and c_prime of those quantities. The exact names are fixed in code, without clashing
    with the existing ones;
  - `k` and `mediator` as declared test arguments.
- **Registry:**
  - the narrow lm handlers come before `lm`;
  - `test-core-registry` checks that a simple, a multiple and a moderation `lm` each match the
    intended handler;
  - a polynomial fit falls back to `lm`.
- **Coverage table:** four new rows that must equal plan §4 rows 79–82. The plan stays the
  specification until the generated table replaces it.
- **The evidence goes in `reprokit/inst/validation/`:**
  - `check-known-answers.R`:
    - `summary(lm)` on mtcars for simple (`mpg ~ wt`), multiple (`mpg ~ wt + hp + factor(am)`) and
      moderation (`mpg ~ wt * hp`);
    - a mediation example whose paths are compared with the three `lm()` fits.
  - `check-range-exactness.R`: entries for t = b/SE, the CI, and F from R².
  - `roundtrip-simulation.R` and `check-designs.R`:
    - k = 1 to 6;
    - factor predictors;
    - weighted fits for the three lm rows;
    - mediation with small and large paths.
  - `check-false-positives.R` (the release gate): correct papers that report adjusted R² as R²,
    unstandardised b labelled β, a subset of terms, and mediation papers that report a·b with a
    bootstrap CI are never flagged as inconsistent. The a·b numbers are never marked as fitting
    or not fitting.
  - `check-planted-errors.R`:
    - a digit changed in b, SE, t or p;
    - an F that does not match R²;
    - df from the wrong k;
    - a mediation path with the wrong df_resid.
- **Plan edits after acceptance:**
  - §5.5 lists `check_lm()` and `check_mediation()`;
  - §13 step 6 points to this ADR;
  - the §4 mediation row: checkable becomes "a, b, c, c′ (b, SE, t, p each)", not checkable adds
    "a·b and its CI (inferred by bootstrap, which resamples cases; summaries cannot)", and the
    choices become "—".

  The other §4 rows are unchanged.
- **Open until implementation:** whether `lm_moderation` should also accept a numeric × factor
  term with more than two levels. In v1 it does not, and such a fit falls back to `lm`.
