# 0001. Package and app in two repositories

Date: 2026-11-22 · Status: accepted (supersedes plan §11 "one repository")

## Context
The plan (§11) proposed one repository with the app in a subfolder. The package must stay
usable, testable and citable without Shiny (§5).

## Decision
Two repositories, `reprokit` and `reprokit-app`, tied together by this workspace repo.

## Consequences
- + The "app contains no statistics" rule is enforced by the boundary: the app can only use
  exported package functions.
- + Package versions and releases are independent of app changes.
- − The app must install the package from a remote: GitHub for CI, **r-universe for webR**
  (shinylive can only load packages that have a WebAssembly build; r-universe builds those).
- − A package change that the app needs = two pull requests, in order.
