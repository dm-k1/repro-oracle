# 0007. `refit()` becomes `reproduce()`; lm covers weights and offsets

Date: 2026-11-22 · Status: accepted (changes plan §5, §8, §12)

## Context
- `refit()` suggested fitting a model again; what the function does is recompute the reported
  numbers from the kit, which is the claim of the whole package.
- Weighted lm and lm with an offset were planned as downgrades, but both are exact from
  summaries.

## Decision
- The verb and the handler field are named `reproduce`.
- The lm handler supports `weights =` and offsets exactly:
  - rows with weight 0 are dropped (as `summary.lm` does);
  - an offset is stored as outcome − offset;
  - with weights the kit stores n, sum of weights, weighted means and weighted centred
    cross-products; unweighted is the special case (sum of weights = n,
    cross-products = (n − 1) × covariance).
- The disclosure check uses unweighted counts of the rows in the fit, so weights cannot hide
  a small cell.

## Consequences
- + The name says what is promised.
- + More real-world lm fits get ✅ without loss of exactness (tested on 8 weighted/offset
  fits plus the random-design simulation).
- − The kit format differs between weighted and unweighted lm (`sum_w` + `sscp` vs `cov`);
  `lm_crossproducts()` hides that from the reproduce code.
