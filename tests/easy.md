# Easy test — Compile and run CRAN volesti

## Goal
Compile and run the CRAN version of `volesti` on macOS (arm64).

---

## What I did

### 1) Installed dependencies in R

```r
install.packages(c("Rcpp","RcppEigen","BH","devtools","testthat","pkgbuild"))
```

---

### 2) Verified toolchain

```r
pkgbuild::check_build_tools(debug = TRUE)
```

---

### 3) Built from CRAN source

Downloaded:

```
volesti_1.1.2-9.tar.gz
```

---

### 4) Installed from the package source directory

```bash
R CMD INSTALL .
```

---

### 5) Ran a basic runtime test

```r
library(volesti)

P <- gen_cube(5, "H")
volume(P)
sample_points(P, 50)
```

---

## Result

- `volesti` loads correctly  
- Volume estimation runs  
- Sampling runs and returns a matrix of expected shape  

---

## Conclusion

The CRAN version of **volesti** compiles and runs successfully on macOS (arm64).
