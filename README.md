# Spectrahedra-R-interface-lab
This repo is a small lab space for my GSoC 2026 work on GeomScale’s “Spectrahedra R-interface” project.

## Progress

- [x] Easy test: built and ran the CRAN `volesti` package on macOS (arm64)
- [x] Hard test: small extension in `Rvolesti` + focused unit test (PR open)


Files:
- `tests/easy.md` — easy test notes (compile + run CRAN volesti)

Plan (short):
- keep notes small and reproducible
- focus on understanding the R ↔ C++ boundary and then implementing the hard test change spectrahedra-lab

# PRs

## GeomScale/volesti
- #439 — https://github.com/GeomScale/volesti/pull/439
- #441 — https://github.com/GeomScale/volesti/pull/441
- #375 — https://github.com/GeomScale/volesti/pull/375

## GeomScale/Rvolesti
- #36 — https://github.com/GeomScale/Rvolesti/pull/36
- Hard test PR: GeomScale/Rvolesti — PR #37
