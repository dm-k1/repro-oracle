# 0018. JARS checked at source: the regression APA sets and where each APA claim comes from

Date: 2026-11-24 · Status: accepted

## Context

1. **ADR 0013 left one item open.** Its Consequences say: "Whether JARS Table 3.1 puts M and SD of
   each variable, or a correlation matrix, in the APA set for regression is not yet checked at
   source. It decides the regression rows, so it is verified before step 4". The plan's rows for
   simple regression, multiple regression and mediation still say "(pending JARS)" (plan §4, lines
   79, 80, 82). Build step 6 waits on this (plan line 252).
2. **The source has now been read.** These are the official JARS–Quant PDFs on apastyle.apa.org
   (© 2024), read as text:
   - Table 1 (general standards), all 3 pages;
   - Table 3 (nonexperimental designs);
   - Table 7 (structural equation modeling);
   - Table 2 was searched for the relevant keywords only, with no hit.

   The Publication Manual (7th ed.) calls the general table "Table 3.1 Quantitative Design
   Reporting Standards (JARS–Quant)". The APA website numbers the same table "Table 1". Not read:
   the Appelbaum et al. (2018) article, which a bot wall blocked. Any difference between its wording
   and the 2024 PDFs is therefore unchecked.
3. **What JARS says on the four open questions:**

   | question | JARS item (Table 1 unless noted) | answer |
   |---|---|---|
   | M and SD of every variable? | "descriptions of each primary and secondary outcome, including the total sample and each subgroup, that includes the number of cases, cell means, standard deviations, and other measures that characterize the data used" | Outcomes only. Predictors are not named. |
   | A correlation matrix? | "structural equation modeling ..., hierarchical linear models, factor analysis, multivariate analyses ... associated variance–covariance (or correlation) matrix or matrices". Table 7 (SEM): "covariance matrix with means, or correlation matrix with standard deviations and means" | Only for those analyses. Plain regression is not named. "Multivariate analyses" could be read to include it, but it is not the usual meaning. |
   | Which test statistics? | "results of all inferential tests conducted, including exact p values ... and reporting the minimally sufficient set of statistics (e.g., dfs, mean square [MS] effect, MS error) needed to construct the tests"; "effect-size estimates and confidence intervals on estimates that correspond to each inferential test conducted, when possible" | Every test with its df, p, effect size and CI. b and SE are not named, but they are the estimate and the statistics that construct its t test. |
   | Category frequencies? | none. Nearest is Table 3: "the number of cases in each group" | Not in JARS. |

4. **What the Manual says (§3.7).** "What constitutes a 'sufficient set of statistics' depends on
   the analyses", and it names "means (or frequencies of cases in each category for a categorical
   variable)" and standard deviations in general terms. For multivariate analyses "(or correlation)
   matrix or matrices often represent a sufficient set of statistics".
5. **Two earlier claims are worded too broadly:**
   - ADR 0010 (and the plan header, line 4) ground the whole APA baseline in "§3.7, §6.43, §6.44,
     §1.14 and JARS Table 3.1". JARS grounds only the inferential tests and the outcome
     descriptives. The variable descriptives and the category frequencies rest on §3.7.
   - JARS's "minimally sufficient set" means the statistics *needed to construct the tests*. That is
     narrower than this project's `sufficient` set, which must determine every reported result
     (ADR 0008, ADR 0013). The two names should not be read as the same thing.

## Decision

**In one line:** JARS asks for every test with its statistics and the outcome descriptives, not
for predictor descriptives or a correlation matrix in regression. The regression APA sets keep the
M and SD of all variables on the strength of Manual §3.7, and say so. The correlations stay missing.

1. **ADR 0013's open item is closed.** JARS does not put a correlation matrix in the APA set for
   simple or multiple regression, moderation or mediation. It does for SEM, HLM and factor analysis
   (Table 1 and Table 7), and those rows are built with their handlers.
2. **The APA set for regression has three parts, each with its own source:**

   | part | source |
   |---|---|
   | per test: b, SE, t, df, p, with the effect size and CI where one exists; the model R² and F | JARS Table 1, "all inferential tests ... minimally sufficient set ... effect-size estimates and confidence intervals" |
   | n, and the M and SD of the outcome | JARS Table 1, "descriptions of each primary and secondary outcome" |
   | the M and SD of each predictor | Manual §3.7, the general means-and-SDs clause, not JARS |

   The `apa` field keeps one set. The source of each part goes in the handler's comment and in
   this ADR, not in the coverage table.
3. **The plan rows lose "(pending JARS)" and stay otherwise as they are** for simple and multiple
   regression:
   - simple regression: missing is still none, because r = b · SD_x / SD_y;
   - multiple regression: missing is still the correlations of all model columns ("coefficients
     depend on how the predictors overlap").
4. **Mediation gets the M and SD of X, M and Y in its APA set**, for the same §3.7 reason as
   regression. Missing becomes the correlations of X, M and Y only. This changes the plan row, which
   until now listed only the paths, a·b, its CI and n.
5. **The χ² category frequencies rest on Manual §3.7**, "frequencies of cases in each category".
   ADR 0016's APA sets are unchanged; only the source of that claim is corrected.
6. **Citation wording from now on:**
   - Write "JARS–Quant (Manual Table 3.1; Table 1 on apastyle.apa.org)", and cite JARS only for
     what point 2 assigns to it.
   - `sufficient` keeps its name. Its basis is Manual §3.7's "sufficient set of statistics", not
     JARS's "minimally sufficient set", which is the narrower "needed to construct the tests".
7. **If the 2018 article's wording differs from the 2024 PDFs** on any item in Context point 3, a
   new ADR records it. Until then the 2024 PDFs are the source.

Alternatives considered:

| option | why not |
|---|---|
| APA set = JARS only (outcome M and SD; predictor M and SD become missing) | Strictly sourced, but the plan's APA set is "APA 7 / JARS" (plan §5.5), and §3.7 asks for means and SDs in general terms. Moving predictor M/SD to missing would tell authors that APA does not ask for them, which overstates JARS's silence. |
| Read "multivariate analyses" as including multiple regression, so the correlation matrix is APA | The usual meaning of multivariate is several outcomes. Putting the correlations in the APA set would make multiple regression look fully reproducible from an APA-complete paper, which is the gap this project exists to show. |
| Keep "pending JARS" until the 2018 article is read | The 2024 PDFs are the current official version and answer every question. Waiting blocks step 6 on a bot wall. |
| Mediation APA set without M and SD of X, M, Y | Inconsistent with regression under the same §3.7 reasoning. |
| Rename `sufficient` to avoid the clash with JARS's "minimally sufficient set" | ADR 0008 chose the name from Manual §3.7. Point 6 removes the ambiguity in the text at no cost to the code. |

## Consequences

- **Plan edits after acceptance** (`docs/project-plan.md`):
  - line 4: the ADR 0010 entry cites JARS for the tests and outcome descriptives only, and points
    to this ADR;
  - line 67: drop "the regression rows are pending verification of JARS Table 3.1 at source (§8)";
  - lines 79 and 80: drop "(pending JARS)";
  - line 82: drop "(pending JARS)", add "M, SD of X, M, Y" to the APA set, and set missing to "the
    correlations of X, M, Y";
  - line 167: "what APA 7 / JARS requires" stays, with a pointer to this ADR for which source covers
    which part;
  - line 252: drop "after verifying JARS Table 3.1 at source".
- **ADR 0013's open item is closed. Step 6 is unblocked** once the factorial ANOVA (ADR 0017) is
  built.
- ADR 0010 and ADR 0016 keep their decisions. Only the sources they cite are corrected, here.
- The regression handlers record in a comment which part of `apa` comes from JARS and which from
  §3.7 (point 2).
- No code changes, and no validation script, because this ADR contains no numerical claim.

## Sources

- JARS–Quant Table 1, general standards (© 2024): https://apastyle.apa.org/jars/quant-table-1.pdf
- JARS–Quant Table 3, nonexperimental designs: https://apastyle.apa.org/jars/quant-table-3.pdf
- JARS–Quant Table 7, structural equation modeling: https://apastyle.apa.org/jars/quant-table-7.pdf
- JARS–Quant Table 2 (keyword search only): https://apastyle.apa.org/jars/quant-table-2.pdf
- APA Publication Manual (7th ed.), §3.7 and Table 3.1.
- Not read: Appelbaum, M., et al. (2018). Journal article reporting standards for quantitative
  research in psychology. *American Psychologist, 73*(1), 3–25.
- ADR 0008, ADR 0010, ADR 0013, ADR 0016.
