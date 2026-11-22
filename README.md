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

## Getting it

`reprokit/` and `reprokit-app/` are **git submodules**: each is its own repository with
its own history, CI and releases, and this repo pins one commit of each.

```bash
git clone --recurse-submodules https://github.com/OWNER/repro-oracle.git
# already cloned without them:
git submodule update --init
```

## Working on it

```bash
Rscript -e 'devtools::check("reprokit")'
Rscript -e 'shiny::runApp("reprokit-app/app")'   # needs reprokit installed
```

## Submodule workflow

A change inside a submodule is committed **in that submodule**, then the new pointer is
committed here:

```bash
git -C reprokit switch main      # then edit, and:
git -C reprokit commit -am "..." && git -C reprokit push
git add reprokit && git commit -m "Bump reprokit" && git push
```

- `git status` here shows `modified: reprokit (new commits)` when the pin is behind.
- `git submodule update --remote` moves both pins to the latest `main` of each repo.
- Submodules check out a detached commit; `git switch main` before you start editing.

## First push

`.gitmodules` points at `https://github.com/OWNER/...`. Replace `OWNER`, create the
three GitHub repositories, then push the submodules **before** this repo (otherwise the
pinned commits don't exist on GitHub yet):

```bash
git -C reprokit     remote add origin https://github.com/OWNER/reprokit.git
git -C reprokit     push -u origin main
git -C reprokit-app remote add origin https://github.com/OWNER/reprokit-app.git
git -C reprokit-app push -u origin main
git remote add origin https://github.com/OWNER/repro-oracle.git && git push -u origin main
```
