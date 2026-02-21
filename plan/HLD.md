# High Level Design — Spectrahedra R Interface
**GSoC 2026 · GeomScale · Rvolesti**

---

`volesti` already knows how to sample from and compute volumes of spectrahedra. The R package (`Rvolesti`) doesn't expose any of it. This project wires the two together.
No new algorithms, no changes to the C++ backend. The hard math is done — we just need the way to use it.

---

## What's a Spectrahedron

A spectrahedron is the feasible set of a semidefinite program (SDP). Formally:

```
S = { x ∈ ℝⁿ  |  A₀ + x₁A₁ + ... + xₙAₙ ⪰ 0 }
```

Each `Aᵢ` is a real symmetric `m × m` matrix. The `⪰ 0` means positive semidefinite.
Two dimensions matter here and they're independent: `n` is the ambient dimension (the number of free variables `x₁...xₙ`), and `m` is the matrix size.

Why care? Spectrahedra show up in quantum information, polynomial optimisation, control theory, and finance. Right now an R user who wants to sample from one has to write their own C++ wrappers or use a completely different tool. That's the gap this project closes.

---

## State of the codebase before this project

A lot of the groundwork is already done

**In `volesti` (C++ — nothing here changes):**

| Component | Header file | Status |
|-----------|-------------|--------|
| `LMI<NT>` | `convex_bodies/spectrahedra/lmi.h` | works |
| `Spectrahedron<NT>` | `convex_bodies/spectrahedra/spectrahedron.h` | works |
| Billiard Walk for spectrahedra | `random_walks/random_walks.hpp` | works |
| Volume via cooling balls | `volume/volume_sequence_of_balls.hpp` | works |

**In `Rvolesti` (this is where all the work goes):**

| Component | File | Status |
|-----------|------|--------|
| `Spectrahedron` R class | `R/spectrahedron.R` | exists |
| `volume()` | `R/volume.R` | missing Spectrahedron branch |
| `sample_points()` | `R/sample_points.R` | missing Spectrahedron branch |
| Rcpp bridge for spectrahedra | `src/` | doesn't exist yet |

The gap is clean. The C++ side works, the R class exists, nothing connects them.

---

## Architecture

Four layers. Each one does exactly one thing.

```
┌──────────────────────────────────────────────────────────┐
│                      R user session                      │
│                                                          │
│    volume(S)          sample_points(S, n, settings)      │
└───────────────────────────┬──────────────────────────────┘
                            │
           ┌────────────────▼──────────────────┐
           │          R dispatch layer         │
           │                                   │
           │   R/volume.R                      │
           │   R/sample_points.R               │
           │                                   │
           │   inherits(P, "Spectrahedron")    │
           │   → validate → parse → Rcpp call  │
           └──────────┬──────────────┬─────────┘
                      │              │
          ┌───────────▼──┐  ┌─────────▼──────────┐
          │ volume_spec  │  │ sample_spec        │
          │ _rcpp()      │  │ _rcpp()            │
          │ [[Rcpp::     │  │ [[Rcpp::           │
          │  export]]    │  │  export]]          │
          └──────┬───────┘  └───────┬────────────┘
                 │                  │
         ┌───────▼───────────────────▼────────────┐
         │       Rcpp bridge layer  (C++)         │
         │                                        │
         │   src/spectrahedron_interface.cpp      │
         │                                        │
         │   R List → std::vector<Eigen::MatrixXd>│
         │   Build LMI<NT> + Spectrahedron<NT>    │
         │   Run walk / volume estimator          │
         │   Return NumericMatrix or double to R  │
         └─────────────────┬──────────────────────┘
                           │
         ┌─────────────────▼────────────────────────────┐
         │     volesti C++ backend (read-only)          │
         │                                              │
         │  LMI<NT> ────────► Spectrahedron<NT>         │
         │                          │                   │
         │            ┌─────────────┴──────────┐        │
         │            │                        │        │
         │  RandomPointGenerator        volume_sequence │
         │  <BilliardWalk, RNG>         _of_balls       │
         └──────────────────────────────────────────────┘
```

---

## Layer 1 — What users see

Nothing changes from the user's perspective. Same functions, same call signature. A user who already calls `volume()` on H-polytopes doesn't need to learn anything new:

```r
A0 <- diag(3)
A1 <- matrix(c(1,0,0, 0,-1,0, 0,0,0), 3, 3)
A2 <- matrix(c(0,1,0, 1,0,0, 0,0,0), 3, 3)
A3 <- matrix(c(0,0,1, 0,0,0, 1,0,0), 3, 3)

S <- Spectrahedron(matrices = list(A0, A1, A2, A3))

# estimate volume
v <- volume(S, settings = list(error = 0.1, random_seed = 42))

# draw samples — returns a dim × n matrix (3 × 500 here)
pts <- sample_points(S, n = 500, settings = list(walk_type = "BiW", walk_length = 5))
```

The dispatch is transparent. Whatever the user passes, the right thing happens behind the scenes.

---

## Layer 2 — R dispatch and validation

`volume()` and `sample_points()` get a new branch near the top of their function bodies. The pattern follows how zonotopes and V-polytopes are already handled in those same files — there are examples right there in the existing source to follow.

Before any Rcpp call is made, two helpers run:

**`validate_spectrahedron(P)`** checks the object structure and the mathematics. In order: the `$matrices` slot exists and is a list of at least 2 matrices; every matrix is numeric, square, and symmetric; all have the same dimensions; and `A₀` is positive definite (Cholesky). If anything fails it throws a descriptive `stop()` before touching C++.

The reason to validate in R rather than letting C++ catch it: Rcpp template error messages are basically unreadable. A clear `stop()` in R beats 200 lines of compiler output every time.

**`parse_*_settings(settings)`** applies defaults, coerces types, and validates ranges. Returns a flat named list ready for Rcpp. Two versions — one for sampling, one for volume, with the volume version extending the sampling one.

---

## Layer 3 — Rcpp bridge

One new file: `src/spectrahedron_interface.cpp`. Two Rcpp exports and three internal helpers.

```cpp
// [[Rcpp::export]]
Rcpp::NumericMatrix sample_spectrahedron_rcpp(
    Rcpp::List    matrices,
    int           dim,
    int           n_samples,
    std::string   walk_type,
    int           walk_length,
    int           random_seed,
    Rcpp::RObject starting_point
);

// [[Rcpp::export]]
double volume_spectrahedron_rcpp(
    Rcpp::List    matrices,
    int           dim,
    double        error,
    std::string   walk_type,
    int           walk_length,
    int           random_seed,
    int           win_len,
    int           n_samples_per_phase,
    Rcpp::RObject starting_point
);
```

`n_threads` is intentionally absent from the bridge signatures. It's accepted at the R level as a reserved field — parallel support is planned post-GSoC. Taking the parameter now and ignoring it in C++ is cleaner than an API break later when it's actually wired up.

The three static helpers shared between both exports:

- **`parse_r_matrices()`** — takes R's list of matrices and returns `std::vector<Eigen::MatrixXd>`. Uses `Eigen::Map` for a zero-copy view of R memory, then pushes a proper deep copy into the vector. The deep copy matters — the walk modifies internal state and we don't want it touching R's memory.
- **`build_spectrahedron()`** — wraps the matrix vector in `LMI<NT>`, wraps that in `Spectrahedron<NT>`, sets the origin as the initial interior point. The origin is guaranteed to be interior when `A₀` is PD (which validation already confirmed), since `M(0) = A₀ ⪰ 0` strictly.
- **`resolve_seed()`** — if `random_seed == 0`, generates a seed from `std::chrono::high_resolution_clock`; otherwise uses the value as-is.

---

## Layer 4 — C++ backend (untouched)

| Component | What it does | Called from |
|-----------|-------------|-------------|
| `LMI<NT>` | Stores `[A₀,...,Aₙ]`, evaluates `M(x)` | `build_spectrahedron()` |
| `Spectrahedron<NT>` | Wraps LMI, provides membership oracle | Both exports |
| `BilliardWalk` | Reflective walk — boundary hits via generalised eigenvalue solve | `sample_spectrahedron_rcpp()` |
| `RDHRWalk` | Random Direction Hit-and-Run | `sample_spectrahedron_rcpp()` |
| `RandomPointGenerator<Walk,RNG>::apply()` | Drives the walk, fills output matrix | `sample_spectrahedron_rcpp()` |
| `volume_sequence_of_balls<Walk,RNG>()` | Cooling balls volume estimator | `volume_spectrahedron_rcpp()` |

No changes to any of these files.

---

## Files — exactly what gets touched

```
Rvolesti/
├── R/
│   ├── volume.R                  ← ADD: Spectrahedron branch
│   ├── sample_points.R           ← ADD: Spectrahedron branch
│   ├── validate_spectrahedron.R  ← CREATE
│   └── settings_parser.R         ← CREATE
├── src/
│   └── spectrahedron_interface.cpp   ← CREATE
├── tests/testthat/
│   └── test_spectrahedron.R      ← CREATE
├── man/
│   ├── volume.Rd                 ← auto-updated via roxygen2
│   └── sample_points.Rd          ← auto-updated via roxygen2
└── NEWS.md                       ← one changelog entry added

```

---

## Settings reference

| Field | Type | Default | Valid values | Applies to |
|-------|------|---------|--------------|-----------|
| `walk_type` | character | `"BiW"` | `"BiW"`, `"RDHR"` | both |
| `walk_length` | integer | `1` | 1–1000 | both |
| `random_seed` | integer | `0` | ≥ 0 (0 = random) | both |
| `n_threads` | integer | `1` | ≥ 1 | both (reserved) |
| `starting_point` | numeric vector | `NULL` | interior point, length = dim | both |
| `error` | numeric | `0.1` | (0, 1) open | `volume()` only |
| `win_len` | integer | `4` | ≥ 1 | `volume()` only |
| `n_samples_per_phase` | integer | `200` | ≥ 10 | `volume()` only |

`n_threads` is accepted and ignored in C++ — that's intentional. BRDHR is not listed because the C++ side isn't ready for it on spectrahedra specifically.

---

## How sampling works

Default walk is Billiard Walk (BiW):

1. Start from `x₀` — origin if `A₀` is PD (always true after validation), or a user-supplied point.
2. Pick a random direction `v` on the unit sphere.
3. Shoot a billiard trajectory from `x₀` in direction `v`. On hitting the spectrahedron boundary, reflect. The boundary hit is found by solving a generalised eigenvalue problem on the LMI matrices — this is the expensive part of each step.
4. After `walk_length` reflections, record the current position as a sample.
5. Repeat from step 2 until `n_samples` points are collected.
6. Return a `dim × n_samples` matrix.

`walk_length = 1` produces the most correlated samples. Higher values reduce correlation at the cost of more computation per sample. For most practical uses `walk_length` between 5 and 20 is a reasonable starting point.

---

## How volume estimation works

Volume uses the Sequence of Balls (SOB) method — also called cooling balls. The direct challenge is that estimating `vol(S)` outright is hard. Estimating ratios between overlapping sets is tractable.

The algorithm constructs an expanding sequence of balls `B₁ ⊂ B₂ ⊂ ... ⊂ Bₖ` where `B₁` is small and contained in `S`, and `Bₖ` fully contains `S`. For consecutive pairs it estimates `vol(Bᵢ ∩ S) / vol(Bᵢ₋₁ ∩ S)` via Monte Carlo — each evaluation uses the membership oracle. Telescoping these ratios gives the full volume:

```
vol(S) ≈ vol(B₁) × [vol(B₂∩S)/vol(B₁∩S)] × ... × [vol(Bₖ∩S)/vol(Bₖ₋₁∩S)]
```

The `error` parameter controls the target relative error. For `n > 10` it's worth increasing `n_samples_per_phase` beyond the default of 200; the estimator needs more samples per phase to achieve the same accuracy in higher dimensions.

---

## Validation rules

Everything is validated in R before Rcpp is touched:

1. `P` has class `"Spectrahedron"`
2. `P$matrices` is a list of length ≥ 2
3. Each matrix is numeric and square
4. Each matrix is symmetric — checked with `isSymmetric()` at `sqrt(.Machine$double.eps)` tolerance
5. All matrices have the same dimensions
6. `A₀` is positive definite (Cholesky — if it succeeds, PD is guaranteed)
7. If `starting_point` is provided: length equals `dim`, and `M(x₀) ⪰ 0` (another Cholesky check)

Errors use 0-based matrix labels (`A0`, `A1`, ...) to match how the mathematics is normally written, even though R uses 1-based indexing internally.

---

## Build

```r
Rcpp::compileAttributes(".")       # registers exports; generates RcppExports.*
devtools::check(args = "--as-cran")
```

The Eigen and volesti headers are already on the include path via the existing `Makevars` — they live in `inst/include/`. No build system changes needed.

---

## Testing plan

Five groups in `tests/testthat/test_spectrahedron.R`:

1. **Validation** — one test per error condition, including the symmetry check
2. **Settings** — correct defaults, type coercion, out-of-range rejection, NA inputs
3. **Sampling** — shape, finite values, reproducibility with same seed, different seeds produce different output, both walk types
4. **Volume** — positive scalar, plausible range against a known reference value (2D ellipse, volume ≈ π), reproducibility
5. **Higher dimensions** — bounded 3D and 5D fixtures; the 5D fixture is constructed carefully to ensure boundedness

---
