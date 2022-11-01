# Making data with ... {#dots-data}



## What's the problem?

A number of functions take `...` to save the user from having to create a vector themselves:

## What are some examples?


```r
sum(c(1, 1, 1))
#> [1] 3
# can be shortened to:
sum(1, 1, 1)
#> [1] 3

f <- factor(c("a", "b", "c", "d"), levels = c("b", "c", "d", "a"))
f
#> [1] a b c d
#> Levels: b c d a
fct_relevel(f, c("b", "a"))
#> [1] a b c d
#> Levels: b a c d
# can be shortened to:
fct_relevel(f, "b", "a")
#> [1] a b c d
#> Levels: b a c d
```

*  `mapply()`



## Why is it important?

In general, I think it is best to avoid using `...` for this purpose because it has a relatively small benefit, only reducing typing by three letters `c()`, but has a number of costs:

*   It can give the misleading impression that other functions in the same
    family work the same way. For example, if you're internalised how `sum()`
    works, you might predict that `mean()` works the same way, but it does
    not:
  
    
    ```r
    mean(c(1, 2, 3))
    #> [1] 2
    mean(1, 2, 3)
    #> [1] 1
    ```
    
    (See Chapter \@ref(dots-position) to learn why this doesn't give an 
    error message.)

*   It makes it harder to adapt the function for new uses. For example, 
    `fct_relevel()` can also be called with a function:
    
    
    ```r
    fct_relevel(f, sort)
    #> [1] a b c d
    #> Levels: a b c d
    ```
    
    If `fct_relevel()` took its input as a single vector, you could easily 
    extend it to also work with functions:
    
    
    ```r
    fct_relevel <- function(f, x) {
      if (is.function(x)) {
        x <- f(levels(x))
      }
    }
    ```
    
    However, because `fct_relevel()` uses dots, the implementation needs to be 
    more complicated:

    
    ```r
    fct_relevel <- function(f, ...) {
      if (dots_n(...) == 1L && is.function(..1)) {
        levels <- fun(levels(x))
      } else {
        levels <- c(...)
      }
    }
    ```

## What are the exceptions?

Note that in all the examples above, the `...` are used to collect a single details argument. It's ok to use `...` to collect _data_, as in `paste()`, `data.frame()`, or `list()`.

## How can remediate it?

If you've already published a function where you've used `...` for this purpose you can change the interface by adding a new argument in front of `...`, and then warning if anything ends up in `...`.


```r
old_foo <- function(x, ...) {
}

new_foo <- function(x, y, ...) {
  if (rlang::dots_n(...) > 0) {
    warning("Use of `...` is now deprecated. Please put all arguments in `y`")
    y <- c(y, ...)
  }
}
```

Because this is a interface change, it should be prominently advertised in packages.

## How can I protect myself?

If you do feel that the tradeoff is worth it (i.e. it's an extremely frequently used function and the savings over time will be considerable), you need to take some steps to minimise the downsides.

This is easiest if you're constructing a vector that shouldn't have names. In this case, you can call `ellipsis::check_dots_unnamed()` to ensure that no named arguments have been accidentally passed to `...`. This protects you against the following undesirable behaviour of `sum()`:


```r
sum(1, 1, 1, na.omit = TRUE)
#> [1] 4

safe_sum <- function(..., na.rm = TRUE) {
  ellipsis::check_dots_unnamed()
  sum(c(...), na.rm = na.rm)
}
safe_sum(1, 1, 1, na.omit = TRUE)
#> Error in `safe_sum()`:
#> ! Arguments in `...` must be passed by position, not name.
#> ✖ Problematic argument:
#> • na.omit = TRUE
```

If you want your vector to have names, the problem is harder, and there's relatively little that you can. You'll need to ensure that all other arguments get a `.` prefix (to minimise chances of a mismatch) and then think carefully about how you might detect problems by thinking about the expect type of `c(...)`. As far as I know, there are no general techniques, and you'll have to think about the problem on a case-by-case basis.

## Selecting variables

A number of funtions in the tidyverse use `...` for selecting variables. For example, `tidyr::fill()` lets you fill in missing values based on the previous row:


```r
df <- tribble(
  ~year,  ~month, ~day,
  2020,  1,       1,
  NA,    NA,      2,
  NA,    NA,      3,
  NA,    2,       1
)
df %>% fill(year, month)
#> # A tibble: 4 × 3
#>    year month   day
#>   <dbl> <dbl> <dbl>
#> 1  2020     1     1
#> 2  2020     1     2
#> 3  2020     1     3
#> 4  2020     2     1
```

All functions that work like this include a call to `tidyselect::vars_select()` that looks something like this:


```r
find_vars <- function(data, ...) {
  tidyselect::vars_select(names(data), ...)
}

find_vars(df, year, month)
#>    year   month 
#>  "year" "month"
```

I now think that this interface is a mistake because it suffers from the same problem as `sum()`: we're using `...` to only save a little typing. We can eliminate the use of dots by requiring the user to use `c()`. (This change also requires explicit quoting and unquoting of `vars` since we're no longer using `...`.)


```r
foo <- function(data, vars) {
  tidyselect::vars_select(names(data), !!enquo(vars))
}

find_vars(df, c(year, month))
#>    year   month 
#>  "year" "month"
```

In other words, I believe that better interface to `fill()` would be:


```r
df %>% fill(c(year, month))
```

Other tidyverse functions like dplyr's scoped verbs and `ggplot2::facet_grid()` require the user to explicitly quote the input. I now believe that this is also a suboptimal interface because it is more typing (`var()` is longer than `c()`, and you must quote even single variables), and arguments that require their inputs to be explicitly quoted are rare in the tidyverse. 


```r
# existing interface
dplyr::mutate_at(mtcars, vars(cyl:vs), mean)
# what I would create today
dplyr::mutate_at(mtcars, c(cyl:vs), mean)

# existing interface
ggplot2::facet_grid(rows = vars(drv), cols = vars(vs, am))
# what I would create today
ggplot2::facet_grid(rows = drv, cols = c(vs, am))
```

That said, it is unlikely we will ever change functions, because the benefit is smaller (primarily improved consistency) and the costs are high, as it impossible to switch from an evaluated argument to a quoted argument without breaking backward compatibility in some small percentage of cases.