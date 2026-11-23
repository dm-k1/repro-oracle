# 0013. Coverage is APA versus sufficient; the v1 scope

Date: 2026-11-23 · Status: accepted

## Context

ADR 0010 gave each handler an `apa` set and a `reproducible` set and drew two outputs from them: an
appendix with "APA plus the gap" and a separate README reporting-gap table, next to the coverage
table. Three things are wrong with that for a class release.

1. **Two tables say one thing.** The coverage table (verdict, what is needed) and the reporting-gap
   table (what APA gives, what to add) both describe the same handler from the same fields, and a
   reader has to combine them to see the point.
2. **The gap is only half the message.** "Add r between conditions" says what to add, not what the
   addition buys. The model the project actually uses has a second question per reported number:
   given the sufficient set, can this APA number be checked, and if not, why not? A mediation
   bootstrap CI is in the APA set and is never checkable from summaries; Pearson's M and SD are
   reported and no relation reaches them. Neither appears in a gap table.
3. **Choices are mixed with numbers.** Student or Welch, the SS type of an unbalanced factorial
   design, centering in a moderation model, tails: these are declarations, not statistics, and the
   §4 table hid them in a "Catch" column.
4. **The scope is too wide to finish.** Plan §12 lists RM ANOVA, SEM, categorical logistic
   regression, standardized effects and Monte Carlo CIs for v1. A class release needs a smaller
   set done fully: share, check and appendix for every test in it.

The word `reproducible` also does not match the vocabulary ADR 0010 itself adopted: APA's term is
"sufficient set of statistics" (§3.7), and `sufficient_stats()` already uses it.

## Decision

**In one line:** per test, *missing = sufficient − APA*; per APA number, *checkable from the
sufficient set, or not, and why*; per test, *the choices to declare*. One generated table and the
appendix both show exactly these.

1. **Rename `reproducible` to `sufficient`** in the registry, the code and the docs. The meaning is
   unchanged: the quantities that determine every reported result, a subset of what `summarise`
   stores (ADR 0010, Consequences).
2. **Two sets and their difference.**
   - `apa` keeps ADR 0010's construction: one function builds it from a small declaration per
     handler, so no handler hand-writes it. The declaration is widened for designs without cells:
     a regression declares its variables and coefficients instead of groups.
   - `sufficient` as above.
   - **Missing** is computed, `setdiff(sufficient, apa)`, never declared. Each missing quantity
     carries a one-line reason, declared once per quantity in the shared vocabulary, so the same
     quantity gets the same reason in every test.
3. **Checkable or not, per APA number.** An APA quantity is *checkable* if the handler's relations
   reach it from the sufficient set (a chain of relations whose inputs are all in `sufficient`). This
   is computed from `relations`, never declared. An APA quantity that no chain reaches is *not
   checkable* and must carry a declared reason (`not_checkable = c(ci_boot = "bootstrap resamples
   cases; summaries cannot")`). A package test fails when an unreached quantity has no reason, and
   when a reason is declared for a quantity the relations do reach.
4. **Choices are a separate field.** `choices` lists the declarations a reader needs that are not
   numbers. It is the union of the variant kinds of the handler's relations (`test`, `tail`,
   `level`, `convention`) and handler-declared extras that no relation varies over (SS type,
   centering). A choice is never listed as a missing number.
5. **`coverage_table()` is the one table.** Generated from the registry, one row per v1 test:

   | test | APA reports | sufficient | missing (reason) | checkable | not checkable (reason) | choices to declare |
   |---|---|---|---|---|---|---|

   `reporting_gap()` is dropped before it is written; the gap is the "missing" column. The triage
   verdict stays an attribute of the handler and is shown in the app and the appendix, not as a
   column here.
6. **`appendix()` has four blocks, then the existing ones.** (a) the APA set, citing §3.7 and §6.43;
   (b) the missing quantities, with their values and reasons; (c) the choices, with what the author
   chose; (d) the APA numbers that stay not checkable, with reasons. Then disclosure, the stated
   reason for not sharing, and what is lost (plan §1, §7).
7. **v1 scope.** Share, check and appendix, each complete, for:

   | family | tests |
   |---|---|
   | χ² | independence, goodness of fit |
   | t | one-sample, independent (Student and Welch), paired |
   | correlation | Pearson |
   | ANOVA | one-way between, factorial between |
   | regression | simple, multiple, moderation, simple mediation |

   Everything else is triage-only in v1 (a verdict and advice, no appendix or check): RM ANOVA,
   logistic and Poisson regression, SEM, Spearman and the rank tests, standalone effect sizes,
   multilevel and mixed models, GEE.

Alternatives considered:

| option | why not |
|---|---|
| Keep two tables (coverage + reporting gap) | Two views of the same fields; the reader has to join them, and they can drift in wording. |
| Declare `missing` per handler | It is a set difference; declaring it allows a missing list that disagrees with `apa` and `sufficient`. |
| Declare checkable per APA number | The relations already say what is reachable; a declaration could claim a check the engine cannot do. Only the *reason* for not checkable is a judgement, so only that is declared. |
| Keep choices in a free-text "catch" column | A choice then cannot be shown in the appendix with the author's answer, and cannot be read by the check side, which already tries the same alternatives as variants. |
| Keep the wide §12 v1 (RM ANOVA, SEM, logistic) | Not finishable as share + check + appendix for every test; a narrower complete v1 teaches the same pattern. |

## Consequences

- **The worked table shown when this was agreed was wrong on two rows, and is corrected here.**
  By ADR 0010's construction rule the APA set contains per-cell n, and for a categorical variable
  the cell frequencies (§3.7: "frequencies of cases in each category"). So for one-way and
  factorial ANOVA the missing set is empty, not "n per group", and for χ² the contingency table is
  in the APA set, so missing is empty there too. The rows where missing is not empty are paired t
  (r between conditions), regression (the correlations among all model variables), moderation (also
  the moments of the product X·Z) and mediation (the correlations among X, M and Y). That is the
  honest form of the project's message: for simple between-group tests APA's generic list already
  suffices; it stops sufficing at the first within-subject or multivariable design.
- Whether JARS Table 3.1 puts M and SD of each variable, or a correlation matrix, in the APA set for
  regression is not yet checked at source. It decides the regression rows, so it is verified before
  step 4 (plan §8: an external source is verified at source resolution before a claim is built on
  it).
- Coverage level 3 now reads "the reported set contains `sufficient`"; the levels themselves
  (ADR 0010 point 2) are unchanged.
- ADR 0010 points 1 (field name `reproducible`) and 5 (the reporting-gap table) are superseded; its
  other points stand. ADR 0009 is unaffected.
- The §4 table in the plan becomes the specification of what `coverage_table()` must generate; once
  the function covers every v1 test, the plan points to its output instead of carrying a hand-made
  copy (reprokit CLAUDE.md).
- Each new handler (χ², the ANOVAs, the regression family) lands with `apa`, `sufficient`,
  `relations`, `not_checkable` and `choices`, plus round-trip and false-positive validation, so the
  table row, the appendix and the check grow together.
- Open, not decided here: checking Pearson's p against the t derived from r needs its own ADR.

## Sources

- APA Publication Manual (7th ed.), §3.7, §6.43, §1.14, as quoted in ADR 0010.
- ADR 0010, ADR 0011, ADR 0012.
