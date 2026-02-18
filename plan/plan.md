# Spectrahedra Support in Rvolesti
Working outline for delivering spectrahedron support without disturbing existing polytope and zonotope users.

## Final Deliverables
- `volume(S)` accepts an S4 `Spectrahedron` (current `matrices` slot only) and returns the standard `list(log_volume, volume)`.
- `sample_points(S, ...)` returns a `d × n` numeric matrix with the same orientation conventions as the existing code paths.
- Legacy signatures, defaults, and compiled symbols remain untouched for non-spectrahedron inputs; old users see the exact behavior they expect.
- Documentation, NEWS, and tests clearly state spectrahedron support and everything passes `R CMD check --as-cran` without warnings.

## Expected Code Changes
- `R/zzz_spectrahedra_dispatch.R` (new) — S3 wrappers that catch spectrahedron objects while delegating other inputs to the existing compiled entry points.
- `src/spectrahedra_rcpp_utils.h` (new) — read/validate the `matrices` slot and construct volesti `Spectrahedron` objects.
- `src/volume_spectrahedra.cpp`, `src/sample_points_spectrahedra.cpp` (new) — C++ glue that exposes the new volume and sampling entry points.
- Supporting edits to `man/sample_points.Rd`, `man/volume.Rd`, `NEWS.md`, and `README.md` for documentation.
- `tests/testthat/test_spectrahedron_dispatch.R` and `tests/testthat/test_spectrahedron_feasibility.R` (new) plus an optional SDPA fixture under `inst/extdata/spectrahedra/` for reusable matrices.

## R Layer Approach
1. Capture the current exported functions:
   ```r
   .sample_points_cpp <- sample_points
   .volume_cpp <- volume
   ```
2. Redefine `sample_points()` and `volume()` as S3 generics that keep the published argument order; add `...` only to guard against stray arguments and fail fast on unknown names.
3. Provide `.default` methods that simply forward to `.sample_points_cpp` and `.volume_cpp` so all non-spectrahedron inputs keep their original code paths.
4. Add `.Spectrahedron` methods that call the new `.Call` entry points with explicit arguments plus a `verbosity` flag restricted to `{0L, 1L, 2L}`; reject other values with a friendly message.

## C++ Bridge Approach
1. **Shared helpers (`spectrahedra_rcpp_utils.h`)**
   - `std::vector<MT> read_lmi_matrices(const Rcpp::Reference&, double tol)` extracts `matrices` from the S4 object and enforces: list length ≥ 2, numeric content, square matrices of identical size, and symmetry within tolerance (throw `Rcpp::exception` with the offending index otherwise).
   - `Spectrahedron<Point> build_spectrahedron(const std::vector<MT>&)` mirrors the conversion already used by `write_sdpa_format_file.cpp`.
2. **Volume entry point (`volume_spectrahedra.cpp`)**
   - Export `// [[Rcpp::export]] SEXP volume_spectrahedra(...)` with the S4 object, algorithm settings, rounding options, and `int verbosity`.
   - Convert the matrices via the helper and feed them to the volesti spectrahedron volume routine. If a requested algorithm is not yet wired, raise a clear error (“CoolingBodies not implemented for Spectrahedra”, etc.).
   - Return a two-element list matching the existing R contract.
3. **Sampling entry point (`sample_points_spectrahedra.cpp`)**
   - Export `// [[Rcpp::export]] NumericMatrix sample_points_spectrahedra(...)` mirroring the R signature; respect RNG seeds via `Rcpp::RNGScope` and manual seeding when needed.
   - Start with the proven random walk (billiard or accelerated billiard) and allow limited overrides through the `random_walk` list. Reject unsupported walks early.
   - Return a matrix with `d` rows and `n` columns, matching the expectations from the polytope path.
4. Run `Rcpp::compileAttributes()` once the files exist so `R/RcppExports.R` and `src/RcppExports.cpp` stay synchronized.

## Testing Strategy
- **Dispatch coverage:** `tests/testthat/test_spectrahedron_dispatch.R` builds a micro spectrahedron (e.g., the 3×3 example from the vignette) and asserts that `sample_points()` returns a matrix and `volume()` responds without noise; initially the expectations can target error messages until the C++ glue lands.
- **Feasibility checks:** `tests/testthat/test_spectrahedron_feasibility.R` loads a small fixture from `inst/extdata`, draws a handful of points with a fixed seed, rebuilds `A0 + x1 A1 + ... + xn An` in R, and confirms the maximum eigenvalue is ≤ `1e-8` to satisfy the negative semidefinite condition. Keep `n ≤ 10` to stay CRAN-friendly, gating heavier runs with `skip_on_cran()` if needed.

## Documentation & Polishing
- Extend the Rd files with a dedicated Spectrahedra section that explains required slots, supported random walks, verbosity, and seed handling.
- Align the `read_sdpa_format_file()` documentation so the described return list matches the actual implementation.
- Add a NEWS bullet announcing spectrahedron support and highlight the example in `README.md`, referencing the Chalkis–Fisikopoulos paper for context.
- Ensure CI covers macOS and Ubuntu with `R CMD check --as-cran` so the spectrahedron path is exercised automatically.

## Rough Schedule
- **Week 1:** Land the R dispatch scaffolding and helper header; confirm existing tests stay green.
- **Week 2:** Implement the C++ entry points and validate manual sampling + volume runs on the tiny example.
- **Week 3:** Add the new `testthat` files plus the `inst/extdata` fixture; tighten failure messaging.
- **Week 4:** Update docs/README/NEWS, regenerate Rd files, and run `R CMD check --as-cran` locally and in CI.
- **Week 5:** Buffer for mentor feedback, optional vignette work, and final polish.
