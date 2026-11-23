# 0012. A reported df stands in for n where it determines n exactly

Date: 2026-11-23 · Status: accepted

## Context

The APA form of a correlation is `r(28) = .41, p = .024`: r, df and p, no n. Before this decision
`check_reported("cor_pearson", r =, df =, p =)` gave level 0, "not checkable". Every relation that
reaches p needs n (t from r and n, then p from t and df), and df was only ever an output
(df from n). The statcheck cross-check worked around it by passing the reader's translation
n = df + 2. The engine did not make it. The same gap exists for a paired or one-sample t reported
as `t(19) = 2.85, p = .010` together with descriptives but without n.

For these handlers df is a fixed function of n with an exact inverse on whole numbers:

- correlation: df = n − 2, so n = df + 2;
- paired t and one-sample t: df = n − 1, so n = df + 1.

## Decision

**A relation may declare an `inverse`. When its single input is not reported and its output is,
`check_reported()` derives the input from the output and uses it like a reported number, except
that nothing is ever checked against it.** In reprokit:

1. `relation(..., inverse =)` takes a one-argument function, and only for a relation with one input.
   `df_from_n` has `inverse = function(df) df + 2` (Pearson) or `df + 1` (paired, one-sample t).
2. `derive_implied_inputs()` (`R/check.R`) maps the rounding interval of the reported output
   through the inverse and snaps it to whole numbers: [ceiling(inv(lower)), floor(inv(upper))].
   A df printed with decimals that contains no whole number implies nothing.
3. The implied input joins the inputs available to the other relations. It is not a reported
   interval, so no relation's output is compared with it, and it is not a descriptive, so a number
   derived from it is not "verified".
4. A relation that would compute the quantity an implied input came from is skipped. `df_from_n`
   would otherwise check df against n = df + 2 and always pass.
5. Implied inputs count as known for the coverage level and for `$missing`. They are listed in
   `$implied_inputs` and printed as "Derived from the reported numbers: n = 30 (from df)."

Pearson r, df and p is therefore level 3. That matches plan §4: for a correlation the APA set is
the whole story.

The independent t gets no inverse. df gives n₁ + n₂ but not the split, and a Welch df is not a
whole number.

Alternatives considered:

| option | why not |
|---|---|
| A plain `n_from_df` relation | df_from_n and n_from_df then check each other: a row that always says "fits" and makes df look verified by itself. |
| Derive n but do not count it for the level | df determines n exactly here, so the reported set carries the same information as one with n. Level 0 or 1 for the standard APA correlation would understate it and push readers to add a redundant n. |
| Leave it to the caller (pass n = df + 2) | Every caller has to know the identity per test. The engine already knows it. |

## Consequences

- The APA correlation string is checkable. `inst/validation/check-statcheck.R` now passes
  r, df and p as printed. Of 591 r(df) strings, 0 are at level 0. The script passes with 0 false
  positives, 0 errors and 0 unexplained statcheck-only errors. Its 13 statcheck-only errors fall
  in the known categories: 9 Welch df, 3 t rounding, 1 one-tailed r.
- `inst/validation/check-planted-errors.R` gains an r/df/p subset with its own base-R oracle.
  Detection on r/df/p:

  | planted error | detected |
  |---|---|
  | decision | 81.9% |
  | p digit | 84.7% |
  | r digit | 69.4% |
  | df from the wrong n | 52.8% (38.9% unevaluable) |

  df from the wrong n is often unevaluable because a wrong df moves the implied n with it. Only p
  can expose it, and only when the shift changes p beyond its rounding interval.
- `inst/validation/check-false-positives.R` stays at 0 false positives, and
  `inst/validation/check-designs.R` adds the r/df/p and r/df/p/CI subsets.
- **New limitation: more numbers can loosen a check.** 3 Pearson papers are flagged on r/df/p but
  not when the printed t is added. p_from_t then uses the printed t (±0.005), which is wider than
  the t derived from r. It is the same family as the Welch df printed with decimals (plan §8).
  check-planted-errors.R counts and prints it instead of failing on it, and the r/df/p oracle
  confirms that each of these papers is detectable. A fix would evaluate p from the derived t
  as well, or from the intersection of both, so a reported number can only narrow a check. That
  belongs in its own decision.
- The mechanism is general. Any future relation from one whole-number input (χ² df from the table
  size, ANOVA df from k) can declare an inverse without new engine code.
