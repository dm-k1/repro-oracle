# Reproducibility oracle — project plan

Status: design decisions up to 24 Nov 2026. Package name: open (working name `reprokit`).
Changed since: repository layout → see `decisions/0001-two-repositories.md`; verdict rule and coverage table (§3, §4, §12) → see `decisions/0004-sufficiency-rule-for-verdicts.md`; framing, checks and appendix (§1, §2, §4, §5.5, §6) → see `decisions/0010-coverage-checks-and-the-apa-gap.md`, which grounds the APA baseline in the Publication Manual (7th ed.) §3.7, §6.43, §6.44, §1.14 and JARS–Quant (Manual Table 3.1) for the tests and outcome descriptives; see `decisions/0018-jars-and-the-regression-apa-sets.md`; coverage model, the one coverage table, the appendix blocks and the v1 scope (§2, §4, §5.5, §6, §12, §13) → see `decisions/0013-coverage-is-apa-versus-sufficient-and-v1-scope.md`, which supersedes ADR 0010 points 1 and 5; the build step 5 handlers (one-way ANOVA, χ², factorial ANOVA; §4, §5.5, §13) → see `decisions/0015-oneway-anova-f-range-and-engine-extensions.md`, `decisions/0016-chi-squared-from-counts.md` and `decisions/0017-factorial-anova-effects-ss-type-and-exact-ranges.md`.

## 1. Purpose and framing

An in-class assignment on data sharing, built around an "oracle": a Shiny app with a solid R package behind it. The app helps researchers make results as reproducible as possible when raw data cannot be shared, and helps readers check whether reported results are consistent with each other.

Core message: **APA already requires you to report a sufficient set of statistics, and says that what counts as sufficient cannot be specified in general terms. This tool specifies it per analysis — and checks whether a paper's numbers meet it.** Sharing the raw data remains the standard (APA §1.14 requires it); when sharing is genuinely impossible, a sufficient set is the minimum you still owe your readers, and the tool names exactly what they lose. The tool's limits are part of the argument: a sufficient set only stays small for simple designs, and the more realistic the design, the closer it comes to the raw data.

Framing rule that follows (`decisions/0010`): the tool never positions itself against APA or beyond it. It implements an APA obligation that APA deliberately left open. Where APA is silent (the paired-t correlation, Welch, sphericity, the d convention), outputs say "APA does not require this" and, where the project demands more, say that this is the project's own rule — never "contrary to APA". Welch versus Student and the d convention are *choices to declare*, not missing numbers (`decisions/0013`).

Link to the source: Chambers (2019), *The Seven Deadly Sins of Psychology*, ch. 4 "The Sin of Data Hoarding". Key points used:

- Most refusals to share are not legal: they are about effort, fear of criticism, and treating data as property.
- Wicherts et al. (2011): reluctance to share is associated with more reporting errors.
- Summary statistics raised suspicion in the Sanna and Smeesters cases; only raw data proved fabrication.
- PLOS audit: about a quarter of sampled papers claimed "all data in the paper" while only giving summaries.
- PRO Initiative: share, or state why not.

The APA Publication Manual (7th ed.) already states the obligation this tool mechanises, and
already uses its vocabulary (`decisions/0010` quotes the passages in full):

- §3.7: a paper must "include sufficient information to help readers fully understand the analyses
  conducted", yet "it is impossible to specify what constitutes a 'sufficient set of statistics' in
  general terms" — the generic list it does give is per-cell n, M and SD (or frequencies).
- §6.43: the data supplied "should allow readers to confirm the basic reported analyses (e.g., cell
  means, standard deviations, sample sizes, correlations)".
- §3.7 also names the deliverable: when a sufficient set is extensive, "the information could be
  supplied in a supplementary data set or an appendix".
- §1.14: "Authors must make their data available after publication", and the Ethics Code (Standard
  8.14) prohibits withholding data from qualified requesters for verification through reanalysis.

So the project's position is not "APA is not enough". It is: APA says a sufficient set must be
reported and that it cannot be specified in general — this package specifies it per analysis, and
checks whether a paper's numbers meet it. The sharpest case is the paired t, where APA's own generic
list (per-cell n, M, SD) does not determine the test statistic.

Risk to design against: the tool must not become a respectable excuse for not sharing. Every output therefore (a) requires a stated reason for not sharing and (b) lists what readers lose.

## 2. Two directions

| Direction | User | Question | Output |
|---|---|---|---|
| Share | author who cannot share raw data | "What must I report so others can reproduce this?" | verdict, markdown appendix in four blocks: (a) the APA set; (b) the missing numbers (sufficient − APA) with their values and a reason each; (c) the choices to declare, with the author's answer; (d) the APA numbers that stay not checkable, with reasons — then disclosure check, reason for not sharing, "what is lost" (`decisions/0013`) |
| Check | reader / reviewer | "Do this paper's reported numbers fit together, and how much of them can be checked at all?" | per reported number: verified / coherent / inconsistent / not checkable, a coverage level, and which missing numbers would raise it — with ranges and variants tried (`decisions/0010`) |

The two directions use the same statistics: what the author's appendix reports is exactly the input the reader's check needs.

## 3. Triage: three verdicts

Whether raw data is "needed" depends on what the reader wants to do, not only on the model. The triage separates exact from approximate reproduction, using one rule (→ `decisions/0004`). An analysis is ✅ only if all three hold: (1) the summaries mathematically determine the result, (2) a round-trip test passes, (3) the summaries are clearly less identifying than the raw data. If one fails, it moves down:

| Verdict | Meaning |
|---|---|
| ✅ Summary statistics suffice | exact reproduction; package generates the markdown appendix |
| 🔁 Synthetic data possible | approximate reproduction only; quality measured by refitting the model on synthetic data; share the raw data if you can |
| 🔒 Needs the raw data | summaries are not sufficient and synthetic data does not work, **or** the needed summaries are close to the raw data (no real privacy gain, only less information). Diagnostics, reanalysis and fraud detection always need the raw data and are always listed under "what is lost" |

Every verdict ends with "sharing the raw data remains the standard", citing APA §1.14 and Ethics Code Standard 8.14 — it is APA policy, not this project's opinion.

## 4. Coverage table (v1 specification)

**In one line:** per test, *missing = sufficient − APA*; per APA number, *checkable from the sufficient set or not, and why*; per test, *the choices to declare* (`decisions/0013`).

This table is the specification `coverage_table()` must generate from the registry. Once the function covers every v1 test, this section points to its output instead of carrying a hand-made copy. Missing and checkable are computed (set difference; reachability through `relations`); only the reasons and the choices are declared. Rows for handlers that do not exist yet state what their relations must reach.

| Test | APA reports | Sufficient | Missing (reason) | Checkable | Not checkable (reason) | Choices to declare |
|---|---|---|---|---|---|---|
| χ² independence | χ², df, p, N, the contingency table, V | the contingency table | — | χ², df, p, V (to build) | — | continuity correction for 2×2 |
| χ² goodness of fit | χ², df, p, N, observed counts | observed counts, H0 proportions | H0 proportions, unless equal ("the expected counts depend on them") | χ², df, p (to build) | — | — |
| One-sample t | n, M, SD, t, df, p, CI, d | n, M, SD, test value | — | df, t, p, CI, d | — | tails, level, d convention |
| Independent t | n, M, SD per group, t, df, p, CI, d | n, M, SD per group | — | df, t, p, CI, d | — | Student/Welch, tails, level, d convention |
| Paired t | n, M, SD per condition, t, df, p, CI, d | n, M, SD per condition, r between conditions | r ("t depends on the within-person correlation; per-condition M and SD do not determine it") | df, t, p, CI, d | — | tails, level, d convention (d_z, d_rm, d_av, g_av) |
| Pearson r | n, M, SD of both variables, r, df, p, CI | n, r | — | df, CI, p (via t from r) | M, SD ("r does not depend on them") | tails, level |
| One-way ANOVA | n, M, SD per group, F, df, p, η² | n, M, SD per group | — | df, F, p, η² | — | effect-size convention; variance assumption (equal variances, classical F, not Welch's) |
| Factorial between | n, M, SD per cell, F, df, p per effect, partial η² | n, M, SD per cell | — | df, F, p, partial η² per effect (to build) | — | SS type for unbalanced designs, effect-size convention |
| Simple regression | n, M, SD of X and Y, b, SE, t, p, R² | n, M, SD of X and Y, r | — (r = b · SD_x / SD_y) | to build | — | — |
| Multiple regression | n, M, SD per variable, b, SE, t, p, R², F | n, M, SD, correlation matrix of all model columns | the correlations ("coefficients depend on how the predictors overlap") | to build | — | factor coding |
| Moderation | as multiple regression | + moments of the product X·Z | the X·Z moments ("cannot be derived from X and Z summaries") | to build | — | centering, coding |
| Simple mediation | a, b, c, c′, a·b, its CI, n, M, SD of X, M, Y | n, M, SD and correlations of X, M, Y | the correlations of X, M, Y | a, b, c, c′, a·b, Sobel (to build) | bootstrap CI ("bootstrap resamples cases; summaries cannot") | CI method |

Triage-only in v1 (a verdict and advice, no appendix or check). Verdicts follow the three-condition rule in §3 (`decisions/0004`); "fails" names the condition that moves a row down.

| Family | Analysis | Verdict | What would be needed | Note |
|---|---|---|---|---|
| Rank tests | Mann-Whitney, Wilcoxon, Kruskal-Wallis, Spearman ρ | 🔒 raw data | ranks | fails 1: means and SDs do not determine rank tests; the ranks are nearly the data |
| GLM | Logistic / Poisson, categorical predictors | ✅ exact (borderline) | counts per combination of predictor levels | outside v1; same disclosure note as χ² |
| | Logistic / Poisson, continuous predictors | 🔁 synthetic | — | fails 1: no finite set of summaries determines the fit |
| | Robust / cluster-robust SEs | 🔒 raw data | residuals | fails 1 |
| Mediation | binary mediator or outcome | 🔁 synthetic | — | fails 1 |
| Effect sizes | standalone β, d, η², ω², g | ✅ only with a declared convention | the statistics they follow from | without a convention, no claim |
| Longitudinal | RM ANOVA, uncorrected F | ✅ exact | M, SD per time point + average covariance | verified: average suffices; outside v1 |
| | RM ANOVA, GG/HF-corrected, or pairwise comparisons | ✅ exact | all pairwise correlations | verified: average changes GG epsilon; outside v1 |
| | Mixed / growth model with missing time points | 🔒 raw data | (per-person sums and cross-products) | fails 3: nearly the raw data |
| | GEE | 🔒 raw data | residuals | fails 1 |
| Multilevel | Linear random intercept | 🔁 synthetic | (n_j, sums and cross-products per cluster) | fails 2: not validated against lme4; v2 |
| | Linear random slopes | 🔒 no claim | — | fails 2: not tested |
| | Logistic multilevel (GLMM) | 🔁 synthetic | — | fails 1 |
| SEM | continuous, ML, complete cases | ✅ exact | covariance matrix, means, n | outside v1 |
| | FIML, MLR / robust, WLSMV | 🔒 raw data | — | fails 1 |
| Every model | diagnostics, reanalysis, fraud detection | 🔒 raw data | — | always listed under "what is lost" |

Pattern for the presentation: for the simple between-group tests APA's generic list already suffices (missing is empty); it stops sufficing at the first within-subject or multivariable design (paired t, regression, moderation, mediation). In longitudinal and multilevel designs the needed summaries approach the raw data, and then they offer no privacy gain.

## 5. Architecture

| Layer | Job | Rule |
|---|---|---|
| R package (core) | triage, sufficient statistics, reproduction, appendices, checks | no UI; fully tested; usable without Shiny |
| Shiny app (front end) | the oracle: forms, buttons, scenario cards | contains no statistics; only calls package functions |
| GitHub | code, tests, documentation, hosting | every push runs the tests |

### 5.1 The sufficient-statistics object (ADR 0008)

One central S3 object for the share direction, made by `sufficient_stats()`, holding:

- verdict (+ reason)
- sufficient statistics (means, covariance matrix, counts, per-cluster summaries)
- metadata: complete-case n, contrast coding, effect-size conventions, rounding precision, SS type
- disclosure flags (small cells, small clusters, extreme values)
- stated reason for not sharing (required)
- "what is lost" list

The share functions create or use this object. The check functions (`check_*()`) do not: they work from the numbers a paper reports. The object stores statistics in that same reporting form (n, M, SD, r) whenever it is sufficient, so the check side sets the vocabulary (`decisions/0009`).

### 5.2 Registry of model handlers

Each analysis type is a handler that defines: how to triage, what to store, how to reproduce, what is lost.

- Adding a model = adding a handler.
- The coverage table in the README/guideline is `coverage_table()`, generated from the registry, so documentation and package cannot contradict each other.

### 5.3 Input interface (decided: both)

- Fitted model objects where possible (standard practice): lm, glm, aov, later lmer / lavaan objects, which retain their model frame.
- Formula + data for tests whose result objects do not contain the data (t.test, chisq.test, cor.test).
- Both routes produce the same object.

### 5.4 Behaviour rules

- `triage()` always returns a verdict, including "summaries insufficient, because…". It never crashes on unsupported models.
- Functions asked to do the impossible (e.g. refit a GLMM from summaries) stop with an error that explains why and points to alternatives (raw data, synthetic data, federated analysis).
- Summaries are computed on complete cases of the model variables only; pairwise deletion is never used.
- Values are stored at full precision.

### 5.5 Planned functions (names provisional)

| Function | Direction | Does |
|---|---|---|
| `triage()` | share | verdict per route, with reasons |
| `sufficient_stats()` | share | stores the sufficient statistics of a fitted model or formula + data |
| `reproduce()` | share | reproduces the results from the sufficient statistics; must equal the original |
| `appendix()` | share | writes the markdown appendix: (a) APA set, (b) missing numbers with values and reasons, (c) choices with the author's answers, (d) APA numbers not checkable with reasons; then disclosure check, reason, what is lost |
| `coverage_table()` | share | the one coverage table (§4), generated from the registry (`decisions/0013`) |
| `synthesize()` | share | (later) synthetic data via synthpop + utility report |
| `check_reported()` | check | the engine: evaluates every relation whose inputs are reported, returns a status per number, a coverage level and what is missing |
| `check_*()` | check | thin wrappers around `check_reported()`, from reported numbers only: `check_t()`, `check_r()` and `check_F()` exist; `check_chisq()` (`decisions/0015`) comes with the χ² handlers. A reported CI is checked inside its test's wrapper, not by a wrapper of its own |

## 6. Check direction: rules

**In one line:** each test is a set of reported quantities plus the equations between them; the check
evaluates every equation whose inputs are present, and says how much of the result that covers
(`decisions/0010`).

- Each handler declares `apa` (what APA 7 / JARS requires; which source covers which part: ADR 0018), `sufficient` (what determines every
  result; renamed from `reproducible`, `decisions/0013`), `relations` (the equations),
  `not_checkable` (a reason per APA number no relation reaches) and `choices` (declarations that are
  not numbers), over one shared vocabulary of quantity names.
- The check is not only about p. Every reported number gets a status: **verified** (recomputed from
  more basic reported numbers, e.g. t from M, SD and n), **coherent** (fits the other derived
  numbers, e.g. p from t and df), **inconsistent**, **not reproducible** (only relations that do not
  falsify reach it, such as a d whose convention is rarely declared, and none fits), or **not
  checkable** (no relation reaches it).
- Coverage level: 0 nothing checkable · 1 internal coherence of statistic, df, p and CI · 2 the
  statistic recomputed from the descriptives · 3 the reported set contains `sufficient`, so the
  result is fully reproducible. The check reports which missing numbers would raise the level.
- Missing inputs are solved for instead of skipped: when the output is reported and one input is
  not, report the range of that input that fits ("t is consistent only if r between conditions lies
  in [.41, .58]"). Inconsistent only when no admissible value works.
- A count that a reported df determines exactly is derived from it when it is not reported: n = df + 2
  for a correlation, n = df + 1 for a paired or one-sample t, so the APA form r(28) = .41, p = .024 is
  checkable. The derived n feeds the other relations and counts for the level; df is never checked
  against it (`decisions/0012`).
- Recompute over every combination of rounding bounds of the reported inputs; the reported statistic must fall inside the resulting range (cf. Nick Brown's `f_range()` logic).
- The reported statistic itself also gets its own rounding range (not a fixed tolerance).
- Try variants before flagging: Student vs Welch, with/without continuity correction, one- vs two-tailed, confidence level (90, 95, 99%), d convention. Report which variant fits. One choice per result: t, df and both CI bounds must fit under the same test and level, not each on its own (`decisions/0014`).
- Check df against n (e.g. n₁ + n₂ − 2) and flag mismatches with possible explanations (exclusions, Welch).
- Check p against the test statistic and df (statcheck rule).
- Wording: "consistent" / "inconsistent with the reported statistics", never "wrong" or "error". Always add: an inconsistency shows the numbers cannot all be correct as reported, not which one is wrong or why.

## 7. Disclosure check

- Summary statistics are not automatically anonymous: small cells, min/max values, rare combinations and differencing (two releases differing by one case) can identify people.
- Flag cells / groups / clusters below a threshold (default n < 5, configurable).
- Describe the check as risk reduction, never as a guarantee of anonymity or legal compliance.
- Applies to synthetic data too (rare combinations and outliers can be reproduced closely). Synthetic files are clearly labelled as fake.

## 8. Validation

- Round-trip tests for every ✅ claim: data → sufficient_stats → reproduce equals base R / car / lavaan / lme4 to a scaled difference < 1e-8 (ADR 0005).
- Random designs: many simulated datasets per handler (unbalanced, missing values, factors, interactions).
- Refusal tests: every 🔁 and 🔒 case returns the right verdict and message.
- Known-answer tests for the check direction: consistent and planted-inconsistent cases. One is real: APA Table 7.8 (`decisions/0010`), a correctly reported table whose t, df and p must never be flagged, and whose stated variant (unequal variance) must be reported as not the one that fits.
- A fixture taken from an external source is verified at source resolution before any claim is built on it. A claim in this repository was published on a digit misread from a low-resolution page image and had to be retracted; interval arithmetic over rounded numbers gives no protection against a wrong input.
- False-positive rate is the headline number, not the test count: over hundreds of random designs per handler, real output rounded as a paper would print it must never be flagged inconsistent. That covers `t.test()` paired, independent (Student with `var.equal = TRUE`, and Welch with its df printed to 1 or 2 decimals), one-sample against `mu = 0` and `mu ≠ 0` (the `mu` argument of `check_reported()`), and `cor.test()`. The draws include the edge cases that once broke the engine: r rounding to 1.000, n of 3 to 6, and SDs reported to zero decimals (`decisions/0011`). Review of the first check engine found 69 false positives in 3600 Pearson checks from one unclamped rounding box, which no unit test had caught.
- Cross-checks with statcheck and scrutiny where they overlap. statcheck has no entry point for already-parsed numbers: the only route is `statcheck(texts, stat = ...)` on APA-formatted strings, so the cross-check synthesises `"t(29) = 2.85, p = .008"` and compares `computed_p`, `error` and `decision_error`. It brackets only the statistic (±.5/10^decimals), then rounds the resulting p bounds to the reported decimals, and ignores df precision — so agreement is expected on `t → p` and divergence wherever a second input is bracketed. statcheck stays in Suggests (its imports include ggplot2, DT, pdftools, rmarkdown and tcltk) and is called only from the validation script, guarded by `requireNamespace()`.
- Standards: rOpenSci statistical software standards, compliance documented with srr; autotest; pkgcheck.
- Validation vignette showing PASS tables (as in the verification scripts made so far).
- **Every numerical claim ships with the script that proves it.** A number quoted in an ADR, the vignette or the README must be reproducible from a committed script in `reprokit/inst/validation/`, following `roundtrip-simulation.R` (header stating the claim and the PASS criterion, a seed, a printed table). Evidence written to a scratch directory and summarised in prose does not count: it has to outlive the conversation that produced it. Current scripts: `roundtrip-simulation.R` (share direction), `check-false-positives.R` (the release gate), `check-range-exactness.R` (interval arithmetic), `check-known-answers.R` (cases with a known answer, including APA Table 7.8).

Verified so far (scripts in the design conversation): independent and paired t, Pearson, one-way ANOVA, lm (slopes, intercept, SEs, R²), χ² and categorical logistic regression from counts, random-intercept likelihood from per-cluster summaries, RM ANOVA F from full vs average covariance and GG epsilon, interaction (incl. centering from the same summaries), mediation a·b, standardized betas, t-test check prototype with rounding ranges.

## 9. Shiny app

Three tabs:

- "I can't share my data" — upload data or pick a demo dataset, choose an analysis, give a reason → verdict, downloadable markdown appendix, "what is lost".
- "Does this paper add up?" — enter reported numbers → consistency verdict with ranges and variants.
- "Class scenarios" — preset fictional study cards, some with planted inconsistencies or weak data-sharing excuses. The opening card is not fictional: APA's own sample t-test table (Manual 7th ed., Table 7.8) reports statistics that all check out, while its note says the t tests assume unequal variance — and the reported t and df fit Student, not Welch. Its Cohen's d also falls outside the range its own descriptives imply, under every convention tried, because the convention is not declared (`decisions/0010`). A card about a table that is right in its numbers and loose in its method is a better opening than a planted error. It makes the point before any invented example does, and it shows why the output never says which number is wrong.

## 10. Hosting (decided: shinylive)

- The app runs entirely in the browser via webR and is hosted free on GitHub Pages.
- Uploaded data never leaves the user's laptop — a deliberate teaching point in a lesson about data protection.
- Constraint: the package must run in webR. Keep the core dependency-light (base R + stats); heavier packages (synthpop, lme4, lavaan) are optional (Suggests). Test their webR availability before relying on them; the synthetic route may stay R-only at first.

## 11. GitHub setup

- ~~One repository: package at the root, Shiny app in a subfolder.~~ Two repositories → `decisions/0001`.
- GitHub Actions: R CMD check + tests on every push; automatic shinylive deployment to GitHub Pages.
- README: generated coverage table, lifecycle badge ("experimental"), prior-art section.

Prior art to cite: statcheck (Nuijten & Epskamp), JATSdecoder `get.stats()` (Böschen), scrutiny (Jung), rsprite2 (Wallrich), unsum/CLOSURE (Jung), Nick Brown's `f_range()`, RIVETS (Brown & Heathers), synthpop (Nowok, Raab & Dibben), faux (DeBruine), DataSHIELD, metacheck (DeBruine, Lakens et al.).

## 12. Scope

| Version | Contents |
|---|---|
| v1 (class) | share, check and appendix, each complete, for: χ² independence and goodness of fit; one-sample, independent (Student and Welch) and paired t; Pearson; one-way and factorial between ANOVA; simple, multiple, moderation and simple-mediation regression (`decisions/0013`). Triage-only (verdict + advice) for everything else: RM ANOVA, logistic/Poisson, SEM, rank tests and Spearman, standalone effect sizes, multilevel and mixed models, GEE. |
| v2 | synthetic route via synthpop with utility report; linear random-intercept model validated against lme4 (✅ only for large clusters); candidates moved out of v1: RM ANOVA, categorical logistic, SEM (ML), Monte Carlo CI as labelled replacement for the bootstrap CI |
| Refuse list (summary route) | GLMMs, logistic/Poisson with continuous predictors, robust/cluster SEs and GEE, mixed models with missing time points, nonparametric tests, bootstrap CIs (Monte Carlo replacement only), SEM with FIML/MLR/WLSMV, random slopes until validated |

## 13. Build order

1. ✅ Package skeleton: sufficient-statistics object, registry, t-test and Pearson handlers, round-trip tests.
2. ✅ Check engine with rounding ranges and variants (`decisions/0009`–`0012`).
3. ✅ `coverage_table()` rewrite on the existing handlers: rename `reproducible` → `sufficient`, add `not_checkable` and `choices`, the computed missing/checkable columns and the package test (`decisions/0013`). Stop for review.
4. ✅ `appendix()` with the four blocks, snapshot tests (the four handlers that existed then: one-sample, independent and paired t; Pearson).
5. χ² (independence, goodness of fit), one-way and factorial ANOVA handlers, each with relations and validation. Done so far: one-way ANOVA and `check_F()` (`decisions/0015`).
6. Regression family (simple, multiple, moderation, mediation).
7. Release v1: README with the generated table, vignette, app share module, NEWS, tag.
8. Shiny app tabs + scenario cards; shinylive deployment via GitHub Actions.
9. (v2) synthetic route; random-intercept model.

## 14. In-class assignment (draft, ~40 min)

1. Scenario cards (5 min) — each group gets a fictional study with a data-sharing constraint; some contain a planted inconsistency.
2. Consult the oracle (15 min) — tab 1 to decide what to share/report, tab 2 to check the card's results.
3. Decide (10 min) — is the restriction legitimate, or one of Chambers' "honest reasons"? Share anyway, and in what form?
4. Plenary (10 min) — compare groups; land the coverage-table pattern: summaries work for simple designs, realistic designs need raw data, so share wherever possible.

## 15. Open decisions

- Package name.
- Default disclosure threshold (5? 10?).
- ~~Which d convention is the default~~ Decided: none. The check tries every convention and reports which fit (`decisions/0014`); the appendix asks the author to declare it as a choice (`decisions/0010`, `decisions/0013`).
- Exact class time available (shapes the scenario set).
