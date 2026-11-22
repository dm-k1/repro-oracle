# 0005. Round-trip tolerance is a scaled difference

Date: 2026-11-22 · Status: accepted (sharpens ADR 0004 condition 2 and plan §8)

## Context
ADR 0004 asks a round-trip to match "to ~1e-8" without saying how the difference is measured.
An absolute difference fails on large numbers that are exact to the last digits: an F of
1e5 carries rounding noise near 1e-11 × 1e5 = 1e-6. A purely relative difference blows up
for numbers near 0 (a p-value of 1e-300, a coefficient of 0).

## Decision
A reproduced number passes when

    |original − reproduced| / max(1, |original|) < 1e-8

That is an absolute difference for numbers up to 1 in size, and a relative difference above
that. Implemented in `roundtrip_diff()` / `expect_reproduces()` and in the validation
simulation.

## Consequences
- + One rule holds for p-values, coefficients and large F alike.
- + Phase B simulation (200 random designs per handler): worst scaled difference
  6.8e-16 (t-test) and 9.8e-13 (lm, with weights and offsets), so the margin is ≥ 4 digits.
- − A tiny number (|x| < 1) is checked only to 1e-8 absolute, not to its own digits. That is
  far below anything a paper reports.
