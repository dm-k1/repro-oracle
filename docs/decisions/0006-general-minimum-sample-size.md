# 0006. A general minimum sample size for every ✅ verdict

Date: 2026-11-22 · Status: accepted (adds to ADR 0004 condition 3)

## Context
ADR 0004 condition 3 is checked per handler (group sizes, cells of 0/1 columns). Some kits
are close to the raw data for a reason no handler-specific check sees: with n = 2, the mean
and SD of a variable give back both values exactly; with n close to the number of stored
statistics, the kit pins the data down almost completely. A rule based on "stored numbers vs
raw values" was rejected: a cross-product matrix Z'Z fixes Z only up to rotation, so counting
numbers does not measure what is identifiable.

## Decision
- Every kit reports its total sample size as `meta$n` (required by `register_handler()`'s
  contract).
- `triage()` downgrades a ✅ verdict to 🔒 when `n < 2 × threshold` (10 at the default
  threshold of 5), before the handler's own disclosure check. A kit without `meta$n` is also
  downgraded: no n, no claim.
- The rule is general code in `triage_with()`, not in each handler, so new handlers get it
  for free.

## Consequences
- + Protects handlers that have no disclosure check of their own.
- + Tied to the same `threshold` the author already sets, so one knob controls both checks.
- − For two-group tests it adds nothing (two groups of ≥ threshold already give
  n ≥ 2 × threshold); it bites for regression and single-sample analyses.
- − The factor 2 is a judgement call, confirmed 2026-11-22: it is the smallest n at which two groups can each reach the threshold, so the general rule matches what the per-group checks already demand of a two-group design.
