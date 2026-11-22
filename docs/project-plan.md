# Reproducibility oracle — project plan

Status: design decisions up to 16 Nov 2026. Package name: open (working name `reprokit`).
Changed since: repository layout → see `decisions/0001-two-repositories.md`; verdict rule and coverage table (§3, §4, §12) → see `decisions/0004-sufficiency-rule-for-verdicts.md`.

## 1. Purpose and framing

An in-class assignment on data sharing, built around an "oracle": a Shiny app with a solid R package behind it. The app helps researchers make results as reproducible as possible when raw data cannot be shared, and helps readers check whether reported results are consistent with each other.

Core message: sharing raw data remains the standard. When sharing is genuinely impossible, here is the minimum you owe your readers, and here is exactly what they lose. The tool's limits are part of the argument: summary statistics only suffice for simple designs, and the more realistic the design, the more you need the raw data.

Link to the source: Chambers (2019), *The Seven Deadly Sins of Psychology*, ch. 4 "The Sin of Data Hoarding". Key points used:

- Most refusals to share are not legal: they are about effort, fear of criticism, and treating data as property.
- Wicherts et al. (2011): reluctance to share is associated with more reporting errors.
- Summary statistics raised suspicion in the Sanna and Smeesters cases; only raw data proved fabrication.
- PLOS audit: about a quarter of sampled papers claimed "all data in the paper" while only giving summaries.
- PRO Initiative: share, or state why not.

Risk to design against: the tool must not become a respectable excuse for not sharing. Every output therefore (a) requires a stated reason for not sharing and (b) lists what readers lose.

## 2. Two directions

| Direction | User | Question | Output |
|---|---|---|---|
| Share | author who cannot share raw data | "What must I report so others can reproduce this?" | verdict, markdown appendix with the needed statistics, disclosure check, "what is lost" |
| Check | reader / reviewer | "Do this paper's reported numbers fit together?" | consistent / inconsistent with the reported statistics, with ranges and variants tried |

The two directions use the same statistics: what the author's appendix reports is exactly the input the reader's check needs.

## 3. Triage: three verdicts

Whether raw data is "needed" depends on what the reader wants to do, not only on the model. The triage separates exact from approximate reproduction, using one rule (→ `decisions/0004`). An analysis is ✅ only if all three hold: (1) the summaries mathematically determine the result, (2) a round-trip test passes, (3) the summaries are clearly less identifying than the raw data. If one fails, it moves down:

| Verdict | Meaning |
|---|---|
| ✅ Summary statistics suffice | exact reproduction; package generates the markdown appendix |
| 🔁 Synthetic data possible | approximate reproduction only; quality measured by refitting the model on synthetic data; share the raw data if you can |
| 🔒 Needs the raw data | summaries are not sufficient and synthetic data does not work, **or** the needed summaries are close to the raw data (no real privacy gain, only less information). Diagnostics, reanalysis and fraud detection always need the raw data and are always listed under "what is lost" |

Every verdict ends with "sharing the raw data remains the standard".

## 4. Coverage table (statistical decisions)

| Family | Analysis | Verdict | What is needed | Catch |
|---|---|---|---|---|
Verdicts follow the three-condition rule in §3 (`decisions/0004`). "Fails" names the condition that moves a row down.

| Family | Analysis | Verdict | What is needed | Catch |
|---|---|---|---|---|
| Group tests | Independent t (Student and Welch), one-way and factorial ANOVA | ✅ exact | n, M, SD per group/cell | SS type (I/II/III) must be recorded for unbalanced designs |
| | One-sample t | ✅ exact | n, M, SD, test value | — |
| | χ² | ✅ exact (borderline) | contingency table of counts | fixed note: "for these variables the table *is* the data"; disclosure check required in the appendix |
| | Nonparametric: Mann-Whitney, Wilcoxon, Kruskal-Wallis | 🔒 raw data | ranks | fails 1: means and SDs do not determine rank tests; the ranks are nearly the data |
| Correlation | Pearson r | ✅ exact | n, r | — |
| | Spearman ρ | 🔒 raw data | ranks | fails 1 (as above) |
| GLM | lm / regression | ✅ exact | n, means and covariance matrix of all model columns (dummies, interactions) | complete cases on model variables only |
| | Interaction / moderation | ✅ exact | covariance matrix including the product column x·z | product moments cannot be derived from x and z; with it, any centering is reproducible afterwards |
| | Logistic / Poisson, categorical predictors | ✅ exact (borderline) | counts per combination of predictor levels | same note and disclosure check as χ² |
| | Logistic / Poisson, continuous predictors | 🔁 synthetic | — | fails 1: no finite set of summaries determines the fit |
| | Robust / cluster-robust SEs | 🔒 raw data | residuals | fails 1: needs per-case residuals |
| Mediation | a, b, c′, indirect a·b, Sobel | ✅ exact | covariance matrix of X, M, Y + n | — |
| | Bootstrap CI | 🔒 not reproducible | raw data + seed | bootstrap resamples cases. A Monte Carlo CI from a, b and their SEs is offered as a **replacement**, labelled as such, not as reproduction |
| | Binary mediator/outcome | 🔁 synthetic | — | fails 1 |
| Effect sizes | β, d, η², partial η², ω², r | ✅ exact, only with a declared convention | follows from the same statistics (β = b · SD_x / SD_y) | d pooled / d_av / d_z, Hedges' g, how an interaction is standardized; without a convention, no claim |
| Longitudinal | Paired t | ✅ exact | M, SD per condition + r between conditions (or M_diff, SD_diff) | r is rarely reported |
| | RM ANOVA, uncorrected F | ✅ exact | M, SD per time point + average covariance | verified: average suffices |
| | RM ANOVA, GG/HF-corrected, or pairwise comparisons | ✅ exact | all pairwise correlations | verified: average changes GG epsilon |
| | Mixed / growth model with missing time points | 🔒 raw data | (per-person sums and cross-products) | fails 3: with few measurements per person these summaries are nearly the raw data |
| | GEE | 🔒 raw data | residuals | fails 1 |
| Multilevel | Linear random intercept | 🔁 synthetic (v1) | (n_j, sums and cross-products per cluster) | fails 2 in v1: likelihood verified, but not validated against lme4. May move to ✅ in v2, only with large clusters (else fails 3) |
| | Linear random slopes | 🔒 no claim | — | fails 2: not tested |
| | Logistic multilevel (GLMM) | 🔁 synthetic | — | fails 1 |
| SEM / path models | continuous variables, ML, complete cases | ✅ exact | covariance matrix, means, n | can be passed to lavaan (sample.cov) |
| | FIML, MLR / robust, WLSMV with ordinal indicators | 🔒 raw data | — | fails 1: FIML needs missingness patterns, MLR needs case-level kurtosis, WLSMV needs polychoric weights from raw data |
| Every model | diagnostics, reanalysis, fraud detection | 🔒 raw data | — | always listed under "what is lost" |

Pattern for the presentation: exact reproduction from summaries works for simple between-group tests and linear models. In longitudinal and multilevel designs, the needed summaries approach the raw data, and then they offer no privacy gain.

## 5. Architecture

| Layer | Job | Rule |
|---|---|---|
| R package (core) | triage, kits, refits, appendices, checks | no UI; fully tested; usable without Shiny |
| Shiny app (front end) | the oracle: forms, buttons, scenario cards | contains no statistics; only calls package functions |
| GitHub | code, tests, documentation, hosting | every push runs the tests |

### 5.1 The kit object

One central S3 object, holding:

- verdict (+ reason)
- sufficient statistics (means, covariance matrix, counts, per-cluster summaries)
- metadata: complete-case n, contrast coding, effect-size conventions, rounding precision, SS type
- disclosure flags (small cells, small clusters, extreme values)
- stated reason for not sharing (required)
- "what is lost" list

All functions create, use or check a kit.

### 5.2 Registry of model handlers

Each analysis type is a handler that defines: how to triage, what to store, how to reproduce, what is lost.

- Adding a model = adding a handler.
- The coverage table in the README/guideline is generated from the registry, so documentation and package cannot contradict each other.

### 5.3 Input interface (decided: both)

- Fitted model objects where possible (standard practice): lm, glm, aov, later lmer / lavaan objects, which retain their model frame.
- Formula + data for tests whose result objects do not contain the data (t.test, chisq.test, cor.test).
- Both routes produce the same kit.

### 5.4 Behaviour rules

- `triage()` always returns a verdict, including "summaries insufficient, because…". It never crashes on unsupported models.
- Functions asked to do the impossible (e.g. refit a GLMM from summaries) stop with an error that explains why and points to alternatives (raw data, synthetic data, federated analysis).
- Summaries are computed on complete cases of the model variables only; pairwise deletion is never used.
- Values are stored at full precision in the kit.

### 5.5 Planned functions (names provisional)

| Function | Direction | Does |
|---|---|---|
| `triage()` | share | verdict per route, with reasons |
| `kit()` | share | builds the kit from a fitted model or formula + data |
| `reproduce()` | share | reproduces the results from a kit; must equal the original |
| `appendix()` | share | writes the markdown appendix (statistics, disclosure check, reason, what is lost) |
| `synthesize()` | share | (later) synthetic data via synthpop + utility report |
| `check_*()` | check | consistency checks for t, F, χ², r, CIs |

## 6. Check direction: rules

- Recompute over every combination of rounding bounds of the reported inputs; the reported statistic must fall inside the resulting range (cf. Nick Brown's `f_range()` logic).
- The reported statistic itself also gets its own rounding range (not a fixed tolerance).
- Try variants before flagging: Student vs Welch, with/without continuity correction, one- vs two-tailed. Report which variant fits.
- Check df against n (e.g. n₁ + n₂ − 2) and flag mismatches with possible explanations (exclusions, Welch).
- Check p against the test statistic and df (statcheck rule).
- Wording: "consistent" / "inconsistent with the reported statistics", never "wrong" or "error". Always add: an inconsistency shows the numbers cannot all be correct as reported, not which one is wrong or why.

## 7. Disclosure check

- Summary statistics are not automatically anonymous: small cells, min/max values, rare combinations and differencing (two kits differing by one case) can identify people.
- Flag cells / groups / clusters below a threshold (default n < 5, configurable).
- Describe the check as risk reduction, never as a guarantee of anonymity or legal compliance.
- Applies to synthetic data too (rare combinations and outliers can be reproduced closely). Synthetic files are clearly labelled as fake.

## 8. Validation

- Round-trip tests for every ✅ claim: data → kit → reproduce equals base R / car / lavaan / lme4 to a scaled difference < 1e-8 (ADR 0005).
- Random designs: many simulated datasets per handler (unbalanced, missing values, factors, interactions).
- Refusal tests: every 🔁 and 🔒 case returns the right verdict and message.
- Known-answer tests for the check direction: consistent and planted-inconsistent cases.
- Cross-checks with statcheck and scrutiny where they overlap.
- Standards: rOpenSci statistical software standards, compliance documented with srr; autotest; pkgcheck.
- Validation vignette showing PASS tables (as in the verification scripts made so far).

Verified so far (scripts in the design conversation): independent and paired t, Pearson, one-way ANOVA, lm (slopes, intercept, SEs, R²), χ² and categorical logistic regression from counts, random-intercept likelihood from per-cluster summaries, RM ANOVA F from full vs average covariance and GG epsilon, interaction (incl. centering from the same kit), mediation a·b, standardized betas, t-test check prototype with rounding ranges.

## 9. Shiny app

Three tabs:

- "I can't share my data" — upload data or pick a demo dataset, choose an analysis, give a reason → verdict, downloadable markdown appendix, "what is lost".
- "Does this paper add up?" — enter reported numbers → consistency verdict with ranges and variants.
- "Class scenarios" — preset fictional study cards, some with planted inconsistencies or weak data-sharing excuses.

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
| v1 (class) | kit/reproduce/appendix for: independent + paired t, one-sample t, Pearson, one-way + factorial ANOVA, RM ANOVA, lm with interactions, χ² / categorical logistic, simple mediation (Monte Carlo CI labelled as replacement for the bootstrap CI), standardized effects with declared convention, SEM (ML, complete cases). Checks for t, F, χ², r, CIs. Triage-only (verdict + advice) for multilevel, GLMM, logistic with continuous predictors, mixed models with missing time points, nonparametric tests, SEM with FIML/MLR/WLSMV. |
| v2 | synthetic route via synthpop with utility report; linear random-intercept model validated against lme4 (✅ only for large clusters) |
| Refuse list (summary route) | GLMMs, logistic/Poisson with continuous predictors, robust/cluster SEs and GEE, mixed models with missing time points, nonparametric tests, bootstrap CIs (Monte Carlo replacement only), SEM with FIML/MLR/WLSMV, random slopes until validated |

## 13. Build order

1. Package skeleton: kit object, registry, t-test and lm handlers, round-trip tests.
2. Remaining v1 handlers + `appendix()` markdown output.
3. `check_*()` functions with rounding ranges and variants.
4. Shiny app (three tabs) + scenario cards.
5. shinylive deployment via GitHub Actions; README with generated coverage table.
6. (v2) synthetic route; random-intercept model.

## 14. In-class assignment (draft, ~40 min)

1. Scenario cards (5 min) — each group gets a fictional study with a data-sharing constraint; some contain a planted inconsistency.
2. Consult the oracle (15 min) — tab 1 to decide what to share/report, tab 2 to check the card's results.
3. Decide (10 min) — is the restriction legitimate, or one of Chambers' "honest reasons"? Share anyway, and in what form?
4. Plenary (10 min) — compare groups; land the coverage-table pattern: summaries work for simple designs, realistic designs need raw data, so share wherever possible.

## 15. Open decisions

- Package name.
- Default disclosure threshold (5? 10?).
- Which d convention is the default, and how users override it.
- Exact class time available (shapes the scenario set).
