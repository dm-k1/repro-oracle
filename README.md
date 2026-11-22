# Reproducibility oracle — workspace

Umbrella for two repositories:

| Folder | Repository | Job |
|---|---|---|
| `reprokit/` | R package | all statistics: triage, kits, refits, appendices, consistency checks |
| `reprokit-app/` | Shiny app (shinylive) | the oracle UI; contains **no statistics**, only calls `reprokit` |

This workspace repo holds what belongs to neither: the project plan and the design decisions.

- [`docs/project-plan.md`](docs/project-plan.md) — the full plan (status 16 Nov 2026)
- [`docs/decisions/`](docs/decisions/) — one short record per decision (ADR format)

> **Package name is still open.** `reprokit` is a working name. Renaming = DESCRIPTION
> `Package:`, the `reprokit::` calls in the app, the folder names, and the r-universe entry.

## Working on it

```bash
# package
cd reprokit && Rscript -e 'devtools::check()'
# app (needs reprokit installed)
cd reprokit-app && Rscript -e 'shiny::runApp("app")'
```

## Turning the sub-repos into submodules

They are currently independent git repos ignored by this one. After creating the three
GitHub repositories:

```bash
cd reprokit      && git remote add origin git@github.com:<you>/reprokit.git     && git push -u origin main
cd reprokit-app  && git remote add origin git@github.com:<you>/reprokit-app.git && git push -u origin main
# then in this workspace repo: remove the two lines from .gitignore, move the folders aside, and
git submodule add git@github.com:<you>/reprokit.git reprokit
git submodule add git@github.com:<you>/reprokit-app.git reprokit-app
```
