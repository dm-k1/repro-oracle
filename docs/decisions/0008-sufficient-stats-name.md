# 0008. `kit()` becomes `sufficient_stats()`; checks do not use it

Date: 2026-11-22 · Status: accepted (changes plan §5, §8, §12, §13)

## Context
- "Kit" says nothing about what the object holds. A reader has to learn the word first.
- The plan said "all functions create, use or check a kit". That is wrong for the check
  direction: `check_*()` works from the numbers a paper reports (t, df, p, rounding), and
  there is no kit when an author shared nothing.

## Decision
- The object holds the **sufficient statistics** of an analysis (= the summaries that
  determine every reported number exactly), so it is named after that:

  | before | after |
  |---|---|
  | `kit()` | `sufficient_stats()` |
  | class `reprokit_kit` | class `reprokit_sufficient_stats` |
  | `new_kit()`, `is_kit()` | `new_sufficient_stats()`, `is_sufficient_stats()` |
  | handler field `kit` | handler field `summarise` |
  | `reproduce(kit)` | `reproduce(s)` |

- The package has two directions that do not share an object:
  - share: `triage()` → `sufficient_stats()` → `reproduce()` / `appendix()`;
  - check: `check_*()` takes reported numbers only.
- Build order is unchanged: all handlers and `appendix()` first, then `check_*()`. (Replaced by ADR 0009: `check_*()` now comes before the remaining handlers.)

## Consequences
- + The function name says what it returns; the docs need no extra term.
- + The check direction is described honestly.
- − Longer name to type.
- Open: the package name `reprokit` (plan §15) no longer matches the object name. It is
  not renamed here; decide together with the final package name.
