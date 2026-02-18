# Easy Test — Compile and Run CRAN volesti
Reproducible notes for the “easy” GSoC qualification task: prove the CRAN release of `volesti` builds and runs on macOS (arm64).

## Environment
- macOS (Apple Silicon) with current Xcode CLT
- R 4.x with `pkgbuild`, `devtools`, and friends pre-installed

## Steps
1. **Install R dependencies**
   ```r
   install.packages(c("Rcpp", "RcppEigen", "BH", "devtools", "testthat", "pkgbuild"))
   ```
2. **Verify toolchain availability**
   ```r
   pkgbuild::check_build_tools(debug = TRUE)
   ```
3. **Download the CRAN source tarball** — `volesti_1.1.2-9.tar.gz`.
4. **Install from source** (run inside the extracted directory):
   ```bash
   R CMD INSTALL .
   ```
5. **Smoke-test runtime**
   ```r
   library(volesti)

   P <- gen_cube(5, "H")
   volume(P)
   sample_points(P, 50)
   ```

## Results
- Package installs cleanly with no compiler warnings beyond the usual Eigen noise.
- `volume(P)` returns a finite value immediately.
- `sample_points(P, 50)` yields a 5 × 50 numeric matrix, matching expectations.

## Conclusion
The CRAN release of **volesti** builds and runs successfully on macOS (arm64), satisfying the easy qualification requirement.
