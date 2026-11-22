# 0002. Host the app with shinylive on GitHub Pages

Date: 2026-11-16 · Status: accepted (plan §10)

## Context
Free hosting, and a lesson about data protection.

## Decision
The app runs in the browser (webR) and is deployed to GitHub Pages by GitHub Actions.

## Consequences
- + Uploaded data never leaves the user's laptop: a teaching point in itself.
- − `reprokit` Imports stay base R + stats; heavy packages (synthpop, lme4, lavaan) are
  Suggests and checked with `requireNamespace()`. Enforced by `tests/testthat/test-dependencies.R`.
- − Every Suggests package must be tested in webR before the app relies on it.
