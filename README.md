# Spectrahedra R Interface Lab
This workspace tracks the exploratory steps for my GSoC 2026 proposal on extending GeomScale’s tooling so that `Rvolesti` can accept spectrahedra alongside the existing polytopes and zonotopes.

## Status Snapshot
- Built and exercised the CRAN `volesti` release locally on macOS (arm64)
- Submitted a small `Rvolesti` extension plus the supporting focused unit test

## Repository Layout
- `plan/plan.md` — end-to-end delivery plan covering deliverables, code touchpoints, validation, and schedule
- `tests/easy.md` — reproducible log for the “easy” GSoC qualification task (compile + run CRAN `volesti`)
- `tests/hard.md` — reproducible log for the “hard” GSoC qualification task
- `README.md` — high-level context, progress tracking, and links to upstream pull requests

## Working Principles
1. Keep notes concise, reproducible, and free of hidden steps.
2. Touch the smallest possible surface in `Rvolesti`: add spectrahedron-specific dispatch rather than rewriting existing polytope code paths.
3. Land tests, docs, and NEWS entries alongside code so `R CMD check --as-cran` stays quiet.

## Upstream Pull Requests
### GeomScale/volesti
- [#439](https://github.com/GeomScale/volesti/pull/439)
- [#441](https://github.com/GeomScale/volesti/pull/441)
- [#375](https://github.com/GeomScale/volesti/pull/375)

### GeomScale/Rvolesti
- [#36](https://github.com/GeomScale/Rvolesti/pull/36)
- [#37](https://github.com/GeomScale/Rvolesti/pull/37)

