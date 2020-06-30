
<!-- README.md is generated from README.Rmd. Please edit that file -->

# libgeos

<!-- badges: start -->

[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
<!-- badges: end -->

The goal of libgeos is to provide access to the
[GEOS](https://trac.osgeo.org/geos/) C API for high-performance geometry
operations within the R package framework. This package contains a copy
of the GEOS sources rather than linking to system GEOS (if it exists),
meaning you don’t have to install anything (other than the package) to
take advantage of GEOS functions in R. It also means you don’t need a
configure script if you’re writing a package that needs GEOS
functionality. Because GEOS is license under the LGPL, dynamically
linking to GEOS (e.g., through the C API exposed in this package) is
generally allowed from a package with any license.

## Installation

You can install the development version of libgeos from
[GitHub](https://github.com/) with:

``` r
# install.packages("remotes")
remotes::install_github("paleolimbot/libgeos")
```

## Example

This package only exists for its exported C API, for which headers are
provided to make calling these functions from Rcpp or another package as
easy and as safe as possible.

``` cpp
#include <Rcpp.h>

// Packages will also need LinkingTo: libgeos
// [[Rcpp::depends(libgeos)]]

// needed in every file that uses GEOS functions
#include "libgeos.h"

// needed exactly once in your package or Rcpp script
// contains all the function pointers and the
// implementation of the function to initialize them
// (`libgeos_init_api()`)
#include "libgeos.c"

// this function needs to be called once before any GEOS functions
// are called (e.g., in .onLoad() for your package)
// [[Rcpp::export]]
void cpp_libgeos_init_api() {
  libgeos_init_api();
}

// regular C or C++ code that uses GEOS functions!
// [[Rcpp::export]]
std::string version() {
  return GEOSversion();
}
```

``` r
cpp_libgeos_init_api()
version()
#> [1] "3.8.1-CAPI-1.13.3"
```

Realistically, you will want to operate on some actual geometry\! It
isn’t possible to link directly to C++ objects in other packages, and
working with the C API can be difficult if you aren’t used to managing
memory yourself. To bridge this gap, the libgeos package provides a
small header-only API that you can use to manage the pointers returned
by functions in the GEOS C API.

``` cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::depends(libgeos)]]
// use this version of the header to use the LibGEOS C++ API
#include "libgeos-rcpp.h" 
#include "libgeos.c"

// [[Rcpp::export]]
void cpp_libgeos_init_api() {
  libgeos_init_api();
}

// [[Rcpp::export]]
NumericVector wkt_area(CharacterVector wkt) {
  NumericVector output(wkt.size());
  
  LibGEOSHandle handle;
  LibGEOSWKTReader reader(handle);
  for (R_xlen_t i = 0; i < wkt.size(); i++) {
    LibGEOSGeometry geom = reader.read(wkt[i]);
    GEOSArea_r(handle.get(), geom.get(), &output[i]);
  }
  
  return output;
}
```

``` r
cpp_libgeos_init_api()
wkt_area("POLYGON ((0 0, 10 0, 10 10, 0 10, 0 0))")
#> [1] 100
```

If you compile the above with `-DLIBGEOS_DEBUG_MEMORY`, you can see that
the C++ wrapper takes care of a lot of object shuffling under the hood\!

``` r
cpp_libgeos_init_api()
wkt_area("POLYGON ((0 0, 10 0, 10 10, 0 10, 0 0))")
#> GEOS_init_r()
#> GEOSWKTReader_create_r()
#> LibGEOSGeometry(GEOSGeometry*)
#> GEOSGeom_destroy_r()
#> GEOSWKTReader_destroy_r()
#> GEOS_finish_r()
#> [1] 100
```

This is especially useful if you’re running code that might error:

``` r
wkt_area("POLYGON (0 0, 10 0, 10 10, 0 10, 0 0))")
#> GEOS_init_r()
#> GEOSWKTReader_create_r()
#> GEOSWKTReader_destroy_r()
#> GEOS_finish_r()
#> Error in wkt_area("POLYGON (0 0, 10 0, 10 10, 0 10, 0 0))"): ParseException: Expected word but encountered number: '0'
```

Other wrappers that might be useful are `LibGEOSWKTWriter`,
`LibGEOSWKBReader`, `LibGEOSWKBWriter`, and `LibGEOSBufferParams`. In
the future, there will probably also be a `LibGEOSSTRtree` to wrap
indexes.
