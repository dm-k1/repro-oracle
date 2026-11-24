# 0016. χ² from counts: table shape, continuity correction, effect size

Date: 2026-11-24 · Status: accepted

## Context

Build step 5 adds χ² of independence and χ² goodness of fit (GOF). ADR 0015 already decided
what χ² shares with ANOVA:
- no CI in its APA set (`ci = FALSE`);
- a variable number of cells;
- p from the statistic, upper tail (`relation_p_from_statistic("chisq")`);
- the wrapper name `check_chisq()`.

The plan §4 rows fix the sets:
- **Independence:** the APA set has χ², df, p, N, the contingency table and V. The sufficient set is
  the table, so nothing is missing (ADR 0013). The choice is the continuity correction for 2×2.
- **GOF:** the APA set has χ², df, p, N and the observed counts. The sufficient set is the counts and
  the H0 proportions. The H0 proportions are missing unless they are equal.

Five things are not yet decided.

1. **The inputs are counts.** A count in a paper is a whole number. So the table that is reported
   determines χ² as one exact value, not as a range. ANOVA has no counterpart to this.
2. **The table has a shape.** Per-cell quantities are flat vectors, but χ² of independence needs the
   r × c shape for its expected counts and its df. The flat vector cannot give the shape: 6 cells
   can be 2 × 3 or 3 × 2 (same df), but 12 cells can be 2 × 6 (df 5) or 3 × 4 (df 6).
3. **The continuity correction.** In a 2 × 2 table, `chisq.test()` applies Yates' correction by
   default and many papers do not. Both are correct analyses that give different χ² and p.
4. **The effect size.** In a 2 × 2 table, Cramér's V equals |φ|; for larger tables it depends on
   min(r, c). φ in a 2 × 2 table can carry a sign. For GOF the usual effect size is Cohen's w.
5. **Test arguments other than mu.** `check_reported()` passes one plain-number argument to the
   relations, `mu`, through `relations_at_mu()`. χ² needs the table shape, and GOF needs the H0
   proportions.

## Decision

**In one line:** χ² is recomputed exactly from the reported counts. The shape and the H0
proportions are declared test arguments. Yates is a falsifying test variant for 2 × 2 tables only.
V and w are conventions that do not falsify.

1. **The counts are the per-cell `n`.** This reuses the existing quantity instead of adding a new
   `count`. A cell frequency is "the n of a cell", and its domain is already whole numbers ≥ 1.
   - The domain becomes ≥ 0 for χ², because a cell can be empty. That is a domain per handler, not
     a change for t or ANOVA.
   - "29.0" rounds to the integer interval [29, 29], like any n. So a reported table always gives a
     point value of χ², and no `range` hook is needed.
   - A table with an expected count of 0 (an empty row or column) is not evaluable.
2. **A new quantity `n_total`** (N, whole number ≥ 1).
   - A relation gives N = Σ n.
   - With it, a paper that reports only χ², df, p, N and V is still checked for coherence (level 1).
   - For t and ANOVA, N stays implied by df and is not added there.
3. **The shape is a test argument, read from a matrix.**
   - `check_chisq(n = matrix(c("12", "30", "25", "8"), nrow = 2), chisq = "...", ...)`: the wrapper
     reads `dim()` before it parses the strings and passes `dims = c(r, c)` on.
   - For GOF, `n` is a vector and `dims` is its length.
   - Without a table, `dims = c(2, 2)` can be given directly; then df and V are checkable from the
     reported χ² and N.
   - df is (r − 1)(c − 1) for independence and k − 1 for GOF. If `dims` is not given, df is not
     checkable, never guessed from the number of cells.
4. **Test arguments in general.** `relations_at_mu(entry, mu)` becomes `relations_at(entry, args)`.
   - `args` is a named list of plain numbers declared per handler: `mu` for the t handlers, `dims`
     for both χ² handlers, `p0` for GOF.
   - An argument that a handler does not declare is an error, as `mu ≠ 0` is now.
   - The t handlers keep their behaviour.
5. **Continuity correction: variant kind `test`, values `pearson` and `yates`, falsifying.**
   - The variant exists only when `dims` is 2 × 2. Larger tables have the Pearson statistic only,
     as in `chisq.test()`.
   - Yates is computed exactly as `chisq.test()` does: |O − E| − min(0.5, |O − E|).
   - χ² and p share one choice through `shared_choice()` (ADR 0014 point 2). A Yates χ² with a
     Pearson p does not fit.
   - It falsifies because both versions are recomputed exactly from the table. A χ² that fits
     neither is an inconsistency, not an unstated convention.
6. **Effect size: quantity `v`, domain [0, 1], kind `convention`, `falsifies = FALSE`.**
   - Independence: V = √(χ² / (N · (min(r, c) − 1))). In a 2 × 2 table this is |φ|.
   - GOF: w = √(χ² / N).
   - V is computed from the uncorrected (Pearson) χ², because that is how it is defined. Two
     relations share the output `v` (ADR 0014 point 4):
     - from the table, which works also when the paper reports a Yates χ²;
     - from χ², N and `dims`, for papers without a table.
   - A reported φ is compared by its size. The check takes the absolute value and says so in the
     print, because the sign only reflects how the rows and columns are ordered.
7. **GOF H0 proportions: test argument `p0`, uniform by default.**
   - `p0` is also a quantity in the sufficient set and not in the APA set, so the coverage table
     lists it as missing. Its reason says: "the expected counts depend on them; not needed when they
     are equal".
   - Given `p0` must have length k and sum to 1 (within 1e-8). Otherwise it is an input error.
8. **Matching and handlers.** There are two handlers, `chisq_independence` and `chisq_gof`, each
   with a declared choice.
   - The share side takes a formula with `data` (plan §5.3):
     - `~ a + b` with `test = "chisq"` for independence;
     - `~ a` with `test = "gof"` and optional `p0` for GOF.
   - `reported()` uses `chisq.test()` (with `correct = TRUE` for 2 × 2, its default).
   - Table and `xtabs` objects as input come later.
9. **Not in v1, and listed as not checkable or as later work:**
   - **G² (likelihood ratio).** An author who reports it writes G² or "likelihood ratio", so it
     is not a variant of χ².
   - **Monte Carlo and Fisher p-values.** They are a different test, chosen when expected counts
     are small.
   - **Tables reported as percentages instead of counts.** The counts are then only known up to
     rounding. χ² is not monotone in the cells, so corners are not exact, and an exact range over
     the integer lattice is a later decision.

Alternatives considered:

| option | why not |
|---|---|
| A new quantity `count` for cell frequencies | Same meaning as per-cell n; a second name splits the vocabulary and the "missing" reasons (ADR 0013 point 2). |
| Infer the shape from df and the number of cells | Ambiguous whenever both factorisations give the same df, and a wrong shape gives wrong expected counts, so it would accuse correct papers. |
| Yates as a convention (`falsifies = FALSE`) | Both versions are exact functions of the table. Not falsifying would let any wrong χ² in a 2 × 2 table pass as "not reproducible". |
| Yates for every table size | `chisq.test()` and the usual practice apply it to 2 × 2 only. More variants lower detection with no correct paper gained. |
| G² as a third `test` variant | Catches the mislabelled G², but costs detection on every table. A paper that reports G² under the name χ² is inconsistent with its own label. |
| Generalise only `mu` to a vector | Keeps one argument name for things that mean different things (a test value, a shape, proportions). |

## Consequences

- The evidence goes in `reprokit/inst/validation/`, not in this ADR:
  - `check-known-answers.R`: Hair × Eye from `HairEyeColor` (4 × 4) and Admit × Gender from
    `UCBAdmissions` (2 × 2, with and without Yates) for independence; `table(mtcars$cyl)` with
    uniform and with given `p0` for GOF. Each is compared with `chisq.test()`.
  - `check-range-exactness.R`: V and w are monotone in χ² and N, so their corners are exact. This
    is one new entry each. χ² from counts is a point value and needs none.
  - `roundtrip-simulation.R` and `check-designs.R` cover tables from 2 × 2 to 4 × 5 and GOF with
    k = 2 to 8, including empty cells.
  - `check-false-positives.R` (release gate): correct papers with Pearson or Yates in 2 × 2, V
    reported as V or as φ, and given or uniform `p0` are never flagged inconsistent.
  - `check-planted-errors.R`: one count changed, a digit in χ² or p, df from the wrong shape, and a
    Yates χ² paired with a Pearson p.
- **Engine changes:**
  - `relations_at_mu()` becomes `relations_at()`, and the registry declares each handler's test
    arguments;
  - new quantities `n_total`, `v` and `p0`;
  - a handler can set the lower domain bound of `n` to 0;
  - `variant_labels()` gets pearson, yates, v and w.
- `check_chisq()` is the first wrapper whose input can be a matrix. Its roxygen must show the 2 × 2
  example.
- The plan rows stay as they are. Plan §5.5 line 159 is updated when `check_chisq()` exists.
- The validity of the χ² approximation (expected counts below 5) is not checked. It goes in the
  handler's `lost` list, not in the check.
