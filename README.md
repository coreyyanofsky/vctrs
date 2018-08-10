
<!-- README.md is generated from README.Rmd. Please edit that file -->

# vctrs

[![Travis build
status](https://travis-ci.org/r-lib/vctrs.svg?branch=master)](https://travis-ci.org/r-lib/vctrs)
[![Coverage
status](https://codecov.io/gh/r-lib/vctrs/branch/master/graph/badge.svg)](https://codecov.io/github/r-lib/vctrs?branch=master)
[![lifecycle](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)

The short-term goal of vctrs specify the behavior of functions that
combine different types of vectors. This will help reason about
functions that combine different types of input (e.g. `c()`, `ifelse()`,
`rbind()`). The vctrs type system encompasses base vectors
(e.g. logical, numeric, character, list), S3 vectors (e.g. factor,
ordered, Date, POSIXct), and data frames; and can be extended to deal
with S3 vectors defined in other packages, as described in
`vignette("extending-vctrs")`.

Understanding and extending vctrs requires some effort from developers,
but it is our hope that the package will be invisible to most users.
Having an underlying theory that describes what type of thing a function
should return will mean that you can build up an accurate mental model
from day-to-day use, and you will be less surprised by new functions.

In the longer-term, vctrs will become the home for tidyverse vector
functions that work with logical and numeric vectors, and vectors in
general. This will make it a natural complement to
[stringr](https://stringr.tidyverse.org) (strings),
[lubridate](http://lubridate.tidyverse.org) (date/times), and
[forcats](https://forcats.tidyverse.org) (factors), and will bring
together various helpers that are currently scattered across packages,
`ggplot2::cut_number()`, `dplyr::coalesce()`, and `tidyr::fill()`. In
the very long-term, vctrs might provide the basis for a [type
system](https://en.wikipedia.org/wiki/Type_system) for vectors that
could help automate documentation and argument checking.

vctrs has few dependencies and is suitable for use from other packages.
(vctrs has a transitional dependency on tibble. Once vctrs is extensible
all tibble related code will move into the tibble package.)

## Installation

vctrs is not currently on CRAN. Install the development version from
GitHub with:

``` r
# install.packages("devtools")
devtools::install_github("r-lib/vctrs")
```

## Motivation

The primary motivation comes from two separate, but related problems.
The first problem is that `base::c()` has rather undesirable behaviour
when you mix different S3 vectors:

``` r
# combining factors makes integers
c(factor("a"), factor("b"))
#> [1] 1 1

# even if you combine with a string
c("a", factor("a"))
#> [1] "a" "1"

# combing dates and date-times give incorrect values
dt <- as.Date("2020-01-1")
dttm <- as.POSIXct(dt)

c(dt, dttm)
#> [1] "2020-01-01"    "4321940-06-07"
c(dttm, dt)
#> [1] "2019-12-31 18:00:00 CST" "1969-12-31 23:04:22 CST"

# as do combining dates and factors: factors
c(dt, factor("a"))
#> [1] "2020-01-01" "1970-01-02"
c(factor("a"), dt)
#> [1]     1 18262
```

This behaviour arises partly because `c()` has dual purposes: as well as
it’s primary duty of combining vectors, it has a secondary duty of
stripping attributes. For example, `?POSIXct` suggests that you should
use `c()` if you want to reset the timezone.

The second problem is that `dplyr::bind_rows()` is not extensible by
others. At the moment it handles S3 classes using a set of heuristics,
but these often fail, and it feels like we really need to think through
the problem in order to build a principled solution. This intersects
with the need to cleanly support more types of data frame columns
including lists of data frames, data frames, and matrices.

## Usage

``` r
library(vctrs)
```

### Base vectors

`vec_c()` works like `c()`, but has stricter coercion rules:

``` r
vec_c(TRUE, 1)
#> [1] 1 1
vec_c(1L, 1.5)
#> [1] 1.0 1.5
vec_c(1.5, "x")
#> Error: No common type for double and character
```

Unlike `c()`, you can optionally specify the desired output class by
supplying a **prototype**, or ptype, for short:

``` r
vec_c(1, 2, .ptype = integer())
#> [1] 1 2
vec_c(1, "x", .ptype = character())
#> [1] "1" "x"
vec_c(1, "x", .ptype = list())
#> [[1]]
#> [1] 1
#> 
#> [[2]]
#> [1] "x"
```

This supports a much wider range of casts (more on that below) than the
automatic coercions, but it can still fail:

``` r
vec_c(Sys.Date(), .ptype = factor())
#> Error: Can't cast date to factor<5152a>
```

### What is a prototype?

Internally, vctrs represents the class of a vector with a 0-length
subset of the vector. This captures all the attributes of the class, and
in many cases you can use existing base functions like (e.g, `double()`,
`factor(levels = c("a", "b"))`). You can use `vec_ptype()` get a concise
summary of the prototype:

``` r
vec_ptype(letters)
#> prototype: character
vec_ptype(1:50)
#> prototype: integer
vec_ptype(list(1, 2, 3))
#> prototype: list
```

Some protoypes have parameters that are also displayed:

``` r
# Factors display a hash of their levels; this lets
# you distinguish different factors at a glance
vec_ptype(factor("a"))
#> prototype: factor<127a2>

# Date times display the timezone
vec_ptype(Sys.time())
#> prototype: datetime<local>

# difftimes display their units
vec_ptype(as.difftime(10, units = "mins"))
#> prototype: difftime<mins>
```

vctrs provides the `unknown()` class to represent vectors of unknown
type:

``` r
vec_ptype()
#> prototype: unknown
vec_ptype(NULL)
#> prototype: unknown

# NA is technically logical, but used in many places to
# represent a missing value of arbitrary type
vec_ptype(NA)
#> prototype: unknown
```

### Coercion and casting

The vctrs type system is defined by two functions: `vec_type2()` and
`vec_cast()`. `vec_type2()` is used for implicit coercions: given two
types, it returns their common type, or an error stating that there’s no
common type. It is commutative, associative, and has identity element,
`unknown()`.

The easier way to explore how coercion works is to give multiple
arguments to `vec_ptype()`. Behind the scenes, it uses `vec_type2()` to
find the common type.

``` r
vec_ptype(integer(), double())
#> prototype: double

# no common type
vec_ptype(factor(), Sys.Date())
#> Error: No common type for factor<5152a> and date
```

`vec_cast()` is used for explicit casts: given a value and a type, it
casts the value to the type or throws an error stating that the cast is
not possible. If a cast is possible in general (i.e. double -\>
integer), but information is lost for a specific input (e.g. 1.5 -\> 1),
it will generate a warning.

``` r
# Cast succeeds
vec_cast(c(1, 2), integer())
#> [1] 1 2

# Cast loses information
vec_cast(c(1.5, 2.5), integer())
#> Warning: Lossy conversion from double to integer [Locations: 1, 2]
#> [1] 1 2

# Cast fails
vec_cast(c(1.5, 2.5), factor("a"))
#> Error: Can't cast double to factor<127a2>
```

The set of possible casts is a subset of possible automatic coercions.
The following diagram summarises both casts (arrows) and coercions
(circles) for all base types supported by vctrs:

![](man/figures/combined.png)

### Factors

Note that the commutativity of `vec_type2()` only applies to the type,
not the parameters of that type. Concretely, the order in which you
concatenate factors will affect the order of the levels in the output:

``` r
fa <- factor("a")
fb <- factor("b")

levels(vec_ptype(fa, fb))
#> NULL
levels(vec_ptype(fb, fa))
#> NULL
```

### Data frames

vctrs defines the type of a data frame as the type of each column
labelled with the name of the column:

``` r
df1 <- data.frame(x = TRUE, y = 1L)
vec_ptype(df1)
#> prototype: data.frame<
#>  x: logical
#>  y: integer
#> >

df2 <- data.frame(x = 1, z = 1)
vec_ptype(df2)
#> prototype: data.frame<
#>  x: double
#>  z: double
#> >
```

The common type of two data frames is the common type of each column
that occurs in both data frame frames, and the union of the columns that
only occur in one:

``` r
vec_ptype(df1, df2)
#> prototype: data.frame<
#>  x: double
#>  y: integer
#>  z: double
#> >
```

Like factors, the order of variables in the data frame is not
commutative, and depends on the order of the inputs:

``` r
vec_ptype(df1, df2)
#> prototype: data.frame<
#>  x: double
#>  y: integer
#>  z: double
#> >
vec_ptype(df2, df1)
#> prototype: data.frame<
#>  x: double
#>  z: double
#>  y: integer
#> >
```

vctrs also knows how to handle data frame and matrix columns:

``` r
df3 <- data.frame(x = 3)
df3$a <- data.frame(a = 2, b = 2)
df3$b <- matrix(c(1L, 2L), nrow = 1)
vec_ptype(df3)
#> prototype: data.frame<
#>  x: double
#>  a: data.frame<
#>     a: double
#>     b: double
#>    >
#>  b: integer[,2]
#> >

df4 <- data.frame(x = 4)
df4$a <- data.frame(a = FALSE, b = 3, c = "a")
df4$b <- matrix(c(3, 5), nrow = 1)
vec_ptype(df4)
#> prototype: data.frame<
#>  x: double
#>  a: data.frame<
#>     a: logical
#>     b: double
#>     c: factor<127a2>
#>    >
#>  b: double[,2]
#> >

vec_ptype(df3, df4)
#> prototype: data.frame<
#>  x: double
#>  a: data.frame<
#>     a: double
#>     b: double
#>     c: factor<127a2>
#>    >
#>  b: double[,2]
#> >
```

### List of

vctr provides a new vector class that represents a list where each
element has the same type. This is an interesting contrast to a data
frame which is a list where each element has the same *length*.

``` r
x1 <- list_of(1:3, 3:5, 6:8)
vec_ptype(x1)
#> prototype: list_of<integer>

# This type is enforced if you attempt to modify the vector
x1[[4]] <- c(FALSE, TRUE, FALSE)
x1[[4]]
#> [1] 0 1 0

x1[[5]] <- factor("x")
#> Error: Can't cast factor<5a425> to integer
```

This provides a natural type for nested data frames:

``` r
vec_ptype(as_list_of(split(mtcars[1:3], mtcars$cyl)))
#> prototype: list_of<data.frame<
#>  mpg : double
#>  cyl : double
#>  disp: double
#> >>
```

## Compared to base R

The following section compares base R and vctrs behaviour.

### Atomic vectors

``` r
# c() will coerce any atomic type to character
c(1, "x")
#> [1] "1" "x"

# vctrs is stricter, and requires an explicit cast
vec_c(1, "x")
#> Error: No common type for double and character

vec_c(1, "x", .ptype = character())
#> [1] "1" "x"
```

### Factors

``` r
fa <- factor("a")
fb <- factor("b")

# c() strips all factor attributes giving an integer vector
# (as documented in ?c)
c(fa, fb)
#> [1] 1 1

# unlist() creates a new factor with the union of the levels 
unlist(list(fa, fb))
#> [1] a b
#> Levels: a b

# vctrs always unions the levels
vec_c(fa, fb)
#> [1] a b
#> Levels: a b
```

### Dates and date-times

``` r
date <- as.Date("2020-01-01")
datetime <- as.POSIXct("2020-01-01 09:00")

# If the first argument to c() is a date, the result is a date
# But the datetime is not converted correctly (the number of seconds
# in the datetime is interpreted as the number of days in the date)
c(date, datetime)
#> [1] "2020-01-01"    "4322088-04-11"

# If the first argument to c() is a datetime, the result is a datetime
# But the date is not converted correctly (the number of days in the
# date is interpreted as the number of seconds in the date)
c(datetime, date)
#> [1] "2020-01-01 09:00:00 CST" "1969-12-31 23:04:22 CST"

# vctrs always returns the same type regardless of the order
# of the arguments, and converts dates to datetimes at midnight
vec_c(datetime, date)
#> [1] "2020-01-01 09:00:00 CST" "2020-01-01 00:00:00 CST"
vec_c(date, datetime)
#> [1] "2020-01-01 00:00:00 CST" "2020-01-01 09:00:00 CST"

# More subtly (as documented), c() drops the timezone, while
# vec_c() preserves it
datetime_nz <- as.POSIXct("2020-01-01 09:00", tz = "Pacific/Auckland")
c(datetime_nz, datetime_nz)
#> [1] "2019-12-31 14:00:00 CST" "2019-12-31 14:00:00 CST"
vec_c(datetime_nz, datetime_nz)
#> [1] "2020-01-01 09:00:00 NZDT" "2020-01-01 09:00:00 NZDT"
```

### Data frames

``` r
df1 <- data.frame(x = TRUE)
df2 <- data.frame(y = 2)

# rbind() requires the inputs to have identical column names
rbind(df1, df2)
#> Error in match.names(clabs, names(xi)): names do not match previous names

# vctrs creates a common type that is the union of the columns
vec_rbind(df1, df2)
#>      x  y
#> 1 TRUE NA
#> 2   NA  2

# Additionally, you can specify the desired output type
vec_rbind(df1, df2, .ptype = data.frame(x = double(), y = double()))
#>    x  y
#> 1  1 NA
#> 2 NA  2

# In some circumstances (combining data frames and vectors), 
# rbind() silently discards data
rbind(data.frame(x = 1:3), c(1, 1000000))
#>   x
#> 1 1
#> 2 2
#> 3 3
#> 4 1
```

### Tibbles

``` r
tb1 <- tibble::tibble(x = 3)

# rbind() uses the class of the first argument
class(rbind(tb1, df1))
#> [1] "tbl_df"     "tbl"        "data.frame"
class(rbind(df1, tb1))
#> [1] "data.frame"

# vctrs uses the common class of all arguments
class(vec_rbind(df1, df1))
#> [1] "data.frame"
class(vec_rbind(tb1, df1))
#> [1] "tbl_df"     "tbl"        "data.frame"
class(vec_rbind(df1, tb1))
#> [1] "tbl_df"     "tbl"        "data.frame"

# (this will help the tidyverse avoid turning data frames
# into tibbles when you're not using them)
```

## Tidyverse functions

There are a number of tidyverse functions that currently need to do type
coercion. In the long run, their varied and idiosyncratic approaches
will be replaced by the systematic foundation provided by vctrs.

``` r
# Data frame functions
dplyr::inner_join() # and friends
dplyr::bind_rows()
dplyr::summarise()
dplyr::mutate()

tidyr::gather()
tidyr::unnest()

# Vector functions
purrr::flatten()
purrr::map_c()
purrr::transpose()

dplyr::combine()
dplyr::if_else()
dplyr::recode()
dplyr::case_when()
dplyr::coalesce()
dplyr::na_if()
```
