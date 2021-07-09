---
title: Non-Standard Evaluation in R
date: 2020-12-23T19:19:39-05:00
keywords: [dev, r]
description: A special and interesting design of R.
draft: false
---

When I first learned R, I was surprised and confused many times by all the
magical things it did. For example, the `lm` function prints not only the
regression results but also the full function call:

```r
lm(Sepal.Length ~ Petal.Length, data = iris)
# Call:
# lm(formula = Sepal.Length ~ Petal.Length, data = iris)
```

If that does not look magical to you, the `subset` function perhaps will
surprise you:

```r
subset(mtcars, cyl > 6)
```

This line runs without any errors in a clean R session, where the `cyl` is
unbound at all in the global environment -- it is one column of the `mtcars`
data frame. So R manages to find a variable that does not exist in the
environment of the function call. How could that be possible? It turns out that
such magical things are all achieved with the special evaluation mechanism of R.

## Lazy evaluation

Probably the term "lazy evaluation" has been mentioned in the first few lectures
of your R course. It basically means that the function arguments in R are not
evaluated immediately but deferred until their values are really needed. So the
canonical example works:

```r
f <- function(x, y) y
f(x, 10)
```

Internally, when R evaluates a function call those arguments are turned into
promises first, as shown in the C `eval` function:

```c
SEXP pargs = promiseArgs(CDR(e), rho);
tmp = applyClosure(e, op, pargs, rho, R_NilValue);
unpromiseArgs(pargs);
```

Be aware that the snippet has been simplified. See the [full definition][eval]
if you are interested. Also note that only calls of closures, which are
basically normal R functions, are processed in this way. Builtin and special
function calls, such as `{`, `+`, etc., are evaluated in a different way.

A promise captures the expression as well as the environment in which the
expression should be evaluated. So in normal cases things will work as expected:

```r
f <- function(x) {
  a <- 1
  b <- 2
  x + a + b
}

a <- 10
b <- 20
f(a + b)
# 33
```

However, R allows us to extract the expression of an argument. Moreover, it also
provides tools to modify and evaluate the expression. That makes it possible to
evaluate arguments in a different way.

## Non-standard evaluation

Non-standard evaluation, or NSE for short, means to evaluate function calls in
a, well, non-standard way. For instance, instead of the value we can just get
the formatted string of an expression:

```r
nse_exprstr <- function(expr) {
  deparse1(substitute(expr))
}
nse_exprstr(x + y * z)
# "x + y * z"
```

Here `substitute` replaces the argument `expr` with its expression and
`deparse1` (new in R 4.0.0) converts the expression to a string. This is how the
base `plot` function generates labels. We can also evaluate an expression in a
different environment from the one that should normally be used:

```r
nse_env <- function(expr) {
  expr <- substitute(expr)
  env <- new.env()
  env$x <- 10
  env$y <- 20
  eval(expr, env = env)
}
nse_env(x + y)
# 30
```

Here we create a new environment and bind two variables `x` and `y` to the
environment. Then the expression is evaluated in the new environment. This is
how the `subset` function achieves its conciseness -- the filtering criteria are
evaluated inside the data frame. Moreover, it is possible to generate code based
on the inputs and evaluate the generated code instead:

```r
lm_mtcars <- function(y) {
  y <- substitute(y)
  lm_expr <- bquote(lm(.(y) ~ cyl + disp, data = mtcars))
  eval(lm_expr)
}

lm_mtcars(mpg)
# lm(formula = mpg ~ cyl + disp, data = mtcars)
lm_mtcars(wt)
# lm(formula = wt ~ cyl + disp, data = mtcars)
```

Here we create an expression for calling the `lm` function with the provided
dependent variable. That is basically how the pipe function `%>%` does it work
(even though it is much more sophisticated). It is also possible to modify the
full function call if that feels like a good idea:

```r
add <- function(x, y) {
  call <- match.call()
  call[[1]] <- as.name("+")
  eval(call)
}
add(1, 2)
# 3
```

NSE is really a powerful tool of R. Many concise and beautiful interfaces are
only possible with NSE. However, that convenience does come with a cost.

## Problems with NSE

If you know Lisp, the above examples may look familiar to you. Indeed they are
similar to Lisp macros. Therefore, functions using NSE have the common problems
of Lisp macros -- they are harder to reason about, are harder to debug, and
generate more obscure error messages than normal functions. Moreover, because
functions with NSE often do both code generation and code evaluation, they may
bring more problems than Lisp macros, which usually only generate code.

If you look at the documentation of `subset`, you will be warned that the
function should be used interactively and may have "unanticipated consequences".
Here is such a case:

```r
mpg <- mtcars$cyl
subset(mtcars, mpg > 10)
mtcars[mpg > 10]
```

The `subset` line gives us a data frame with rows of `mpg > 10`, while the `[`
line gives us an empty data frame. Because the expression `mpg > 10` is
evaluated in the data frame in the first case, the variable `mpg` is shadowed by
the column in the data frame. That forces us to choose a different name:

```r
mpg2 <- mtcars$cyl
subset(mtcars, mpg2 > 10)
```

It shows that with NSE variables can be shadowed unexpectedly -- the hygiene
problem. Even worse, there is no easy way to tell if a function uses NSE. If the
author of a function does not write anything on that, there probably would be
lots of fun to figure it out.

Therefore, I believe NSE should be only used sparingly. Even though NSE may
bring a friendly interface at the beginning, it could cause more troubles later.
I think `dplyr` is an example for that. It employs NSE heavily so users rarely
need to write the name of a data frame. However, when people start to write
wrappers around it, strange errors occur. Now it uses "tidy evaluation" to
address that issue, but in my opinion that is hard to understand for beginners,
although I am a happy user of `dplyr`. So please do think twice before using
NSE. If you believe that it pays off, be sure to document it carefully.

[eval]: https://github.com/wch/r-source/blob/94d52963eed43abd7fabdade683e84d788cc4cf7/src/main/eval.c#L641
