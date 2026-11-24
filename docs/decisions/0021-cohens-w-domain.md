# 0021. Cohen's w has no upper bound of 1

Date: 2026-11-24 · Status: accepted

## Context

- ADR 0016 point 6 gives the effect-size quantity `v` the domain [0, 1] and uses it for both χ²
  handlers:
  - Cramér's V for independence;
  - Cohen's w for goodness of fit (GOF).
- V is bounded by 1, but w is not. w = √(χ² / N), and for k categories its maximum is √(k − 1).
  That maximum is reached when every case falls in one category under uniform H0 proportions.
- A correct GOF paper with k ≥ 3 and a large effect can report w > 1. A domain of [0, 1] would
  reject that number as impossible. This breaks the rule that ambiguity resolves towards silence,
  and it would fail the false-positive gate.
- The χ² implementation (build step 5a) brought this to light.

## Decision

**In one line:** `v` keeps [0, 1] for independence and gets [0, ∞) for GOF, through the per-handler
domain override that ADR 0016 point 1 already introduced for `n`.

- `chisq_independence`: `v` domain [0, 1], unchanged.
- `chisq_gof`: `v` domain [0, ∞).
  - The bound √(k − 1) is not used as an upper domain. It depends on `p0`: it is larger under
    unequal H0 proportions.
  - A reported w that is too large is caught by the relation, not by the domain.

Alternatives considered:

| option | why not |
|---|---|
| A separate quantity `w` | ADR 0016 point 6 deliberately shares one output `v` with variant labels v and w. A second name splits the vocabulary for one concept, effect size from χ². |
| Domain [0, √(k − 1)] | The bound holds only for uniform `p0`. Under unequal `p0` it would reject correct values. |

## Consequences

- ADR 0016 point 6 is amended for GOF only. Its other points stand.
- `check-false-positives.R` includes a GOF case with w > 1.
