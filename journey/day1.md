# Day 1 — Environment setup + first run (macOS)

## System
- macOS: 26.1 (arm64)
- R: 4.5.2 (Homebrew)
- Xcode CLT: /Library/Developer/CommandLineTools
- Git configured:
  - user.name = Mohit-Lakra
  - user.email = mohitlakra2007777@gmail.com

## R build tools check
Ran `pkgbuild::check_build_tools(debug = TRUE)` and it successfully compiled a small C file.

## Install (from source)
Downloaded CRAN source: `volesti_1.1.2-9.tar.gz` and built locally.

## Quick smoke test in R
```r
library(volesti)
P <- gen_cube(5, "H")
v <- volume(P)
S <- sample_points(P, 50)
print(v)
print(dim(S))
```

### Output
```
Loading required package: Rcpp
$log_volume 
[1] 3.337297

$volume 
[1] 28.14296 

[1] 5 50
```
