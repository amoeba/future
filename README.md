# future: A Future for R

## Introduction
In programming, a _future_ is an abstraction for a _value_ that may be available at some point in the future.  The state of a future can either be _unresolved_ or _resolved_.  As soon as it is resolved, the value is available instantaneously.  If the value is queried while the future is still unresolved, the current process is _blocked_ until the future is resolved.  Exactly how and when futures are resolved depends on what strategy is used to evaluate them.  For instance, a future can be resolved using a "lazy" strategy, which means it is resolved only when the value is requested, if at all.  Another approach is an "eager" strategy, which means that it starts to resolve the future as soon as it is created.  Yet other strategies may to resolve futures asynchronously, for instance, by evaluating expressions concurrently on a compute cluster.

### Futures in R

The purpose of the 'future' package is to define and provide a minimalistic Future API for R.  The package itself only provides mechanisms for evaluating expressions _synchronously_ via "lazy" and "eager" futures.  More advanced strategies will be implemented by other packages extending the 'future' package.  For instance, the 'async' package resolves futures _asynchronously_ via any of the many backends that the '[BatchJobs]' framework provides, e.g. multicore processing on a single machine, or distributed on a compute cluster via a job queue.  The lazy and the eager futures provided by this package exist mainly for the purpose of illustrating how futures work and for troubleshooting code that uses other types of futures but for some reason fail when being resolved.

Here is an example illustrating how to create and resolve a future:

```r
> library(future)
> f <- future({
+   message("Resolving...")
+   3.14
+ })
Resolving...
> v <- value(f)
> v
[1] 3.14
```
Note how the future is resolved as soon as we create it using `future()`.  This is because the default strategy for resolving futures in the 'future' package is to evaluate them in an "eager" and synchronous manner, which emulates how R itself typically evaluates expressions, cf.
```r
> v <- {
+   message("Resolving...")
+   3.14
+ }
Resolving...
> v
[1] 3.14
```

We can switch to  a "lazy" evaluation strategy using the `plan()` function, e.g.

```r
> plan(lazy)
> f <- future({
+   message("Resolving...")
+   3.14
+ })
> v <- value(f)
Resolving...
> v
[1] 3.14
```

In this case the future is unresolved until the point in time when we first ask for its value (which also means that a lazy future may never be resolved).


### Promises of successful futures
An important part of a future is the fact that, although we do not necessarily control _when_ a future is resolved, we do have a "promise" that it _will_ be resolved (at least if its value is requested).  In other words, if we ask for the value of a future, we are guaranteed that the expression of the future will be evaluated and its value will be returned to us (or an error will be generated if the evaluation caused an error).  An alternative to a `future-value` pair of function calls is to use the `%<=%` infix assignment operator (also provided by the 'future' package).  For example,

```r
> plan(lazy)
> v %<=% {
+   message("Resolving...")
+   3.14
+ }
> v
Resolving...
[1] 3.14
```

This works by (i) creating a future and (ii) assigning its value to variable `v` as a _promise_.  Specifically, the expression/value assigned to variable `v` is promised to be evaluated/resolved (no later than) when it is requested.  Promises are built-in constructs of R (see `help(delayedAssign)`).



### Eager, lazy and parallel futures
You are responsible for your own futures and how you choose to resolve them may differ depending on your needs and your resources.  The 'future' package provides two evaluation strategies for futures, namely "lazy" and "eager", implemented by functions `lazy()` and `eager()`.  Although not implemented by this package, other R packages may provide strategies for "parallel futures" that are evaluated asynchronously on, for instance, a compute cluster.  Since an asynchronous strategy is more likely to be used in practice, the built-in eager and lazy mechanisms try to emulate those as far as possible while still evaluating them in a _synchronous_ way.

For instance, the default is that the future expression is evaluated in _a local environment_ (cf. `help("local")`), which means that any assignments are done to local variables only - such that the environment of the main/calling process is unaffected.  Here is an example:

```r
> a <- 2.71
> x %<=% { a <- 3.14 }
> x
[1] 3.14
> a
[1] 2.71
```
This shows that `a` in the calling environment is unaffected by the expression evaluated by the future.  If needed, it is possible to evaluate both lazy and eager futures in the calling environment (but the global variables cannot be "frozen").  For instance,
```r
> plan(lazy, local=FALSE, globals=FALSE)
> a <- 2.71
> x %<=% { a <- 3.14 }
> a
[1] 2.71
> x
[1] 3.14
> a
[1] 3.14
```

### Different strategies for different futures
Sometimes one may want to use an alternative evaluation strategy for a specific future.  Although one can use `old <- plan(new)` and afterward `plan(old)` to temporarily switch strategies, a simpler approach is to use the `%plan%` operator, e.g.
```r
> plan(eager)
> a <- 0
> x %<=% { 3.14 }
> y %<=% { a <- 2.71 } %plan% lazy(local=FALSE, globals=FALSE)
> x
[1] 3.14
> a
[1] 0
> y
[1] 2.71
> a
[1] 2.71
```
Above, the expression for `x` is evaluated eagerly (in a local environment), whereas the one for `y` is evaluated lazily in the calling environment.


### Nested futures
It is possible to nest futures in multiple levels and each of the nested future may be resolved using a different strategy, e.g.
```r
> plan(lazy)
> c %<=% {
+   message("Resolving 'c'")
+   a %<=% { 
+     message("Resolving 'a'")
+     3
+   } %plan% eager
+   b %<=% {
+     message("Resolving 'b'")
+     -9 * a 
+   }
+   message("Local variable 'x'")
+   x <- b / 3
+   abs(x)
+ }
> d <- 42
> d
[1] 42
> c
Resolving 'c'
Resolving 'a'
Local variable 'x'
Resolving 'b'
[1] 6
```

## Assigning futures to environments and list environments
The `%<=%` assignment operator _cannot_ be used in all cases where the regular `<-` assignment operator can be used.  For instance, it is _not_ possible to assign future values to a _list_;

```r
> x <- list()
> x$a %<=% { 2.71 }
Error: Subsetting can not be done on a 'list'; only to an environment: 'x$a'
```

This is because _promises_ themselves cannot be assigned to lists.  More precisely, the limitation of future assignments are the same as those for assignments via the `assign()` function, which means you can only assign _future values_ to environments (defaulting to the current environment) but nothing else, i.e. not to elements of a vector, matrix, list or a data.frame and so on.  To assign a future value to an environment, do:

```r
> env <- new.env()
> env$a %<=% { 1 }
> env[["b"]] %<=% { 2 }
> name <- "c"
> env[[name]] %<=% { 3 }
> as.list(env)
$a
[1] 1

$b
[1] 2

$c
[1] 3
```

If _indexed subsetting_ is needed for assignments, the '[listenv]' package provides _list environments_, which technically are environments, but at the same time emulate how lists can be indexed.  For example,
```r
> library(listenv)
> x <- listenv()
> for (ii in 1:3) {
+   x[[ii]] %<=% { rnorm(ii) }
+ }
> names(x) <- c("a", "b", "c")
```
Future values of a list environment can be retrieved individually as `x[["b"]]` and `x$b`, but also as `x[[2]]`, e.g.
```r
> x[[2]]
[1] -0.6735019  0.9873067
> x$b
[1] -0.6735019  0.9873067
```
Just as for any type of environment, all  values of a list environment can be retrieved as a list using `as.list(x)`.  However, remember that future assignments were used, which means that unless they are all resolved, the calling process will block until all values are available.


## Failed futures
Sometimes the future is not what you expected.  If an error occurs while evaluating a future, the error is propagated and thrown as an error in the calling environment _when the future value is requested_.  For example, 
```r
> plan(lazy)
> f <- future({ 
+   stop("Whoops!")
+   42
+ })
> value(f)
Error in eval(expr, envir, enclos) : Whoops!
```
The error is thrown each time the value is requested, that is, trying to get the value again will generate the same error:
```r
> value(f)
Error in eval(expr, envir, enclos) : Whoops!
```

Exception handling of future assignments via `%<=%` works analogously, e.g.
```r
> plan(lazy)
> x %<=% ({ 
+   stop("Whoops!")
+   42
+ })
> y <- 3.14
> y
[1] 3.14
> x
Error in eval(expr, envir, enclos) : Whoops!
> x
Error in eval(expr, envir, enclos) : Whoops!
In addition: Warning message:
restarting interrupted promise evaluation
```
That latter warning is from R itself, notifying us that it already tried to evaluate the promise and tried another time.

The provided "eager future" is very special in the sense that it is resolved immediately.  More specifically, the expression is evaluated _before the future itself is created_.  Because of this, the value of an "eager future" can never throw an error; if an error would occur, it would have prevented the future from being created in the first place, and without the future the corresponding future value/promise will also not exist.  For example,
```r
> plan(eager)
> x %<=% ({
+   a <- 3.14
+   stop("Whoops!")
+   42
+ })
Error in eval(expr, envir, enclos) : Whoops!
> x
Error: object 'x' not found
```

## Globals
The 'future' package does not provide mechanisms for controlling how global variables and functions ("globals") are resolved.  Instead, this important task is passed on to the mechanism that evaluates the future expressions(*).  For instance, concurrent evaluation on compute clusters requires that globals are properly identified and exported to each compute node.  Having said this, the `lazy()` function, which implements lazy futures, does indeed look for globals and "freezes" them at the time point when the future is created.  This assures that the result of a lazy future will be the same regardless of when it is resolved, e.g. before or after globals change.  Since eager futures are resolved upon creation, any globals will also be resolved at this time and therefore there is no need for the `eager()` function to handle globals specifically.  If you wish to implement your own future mechanism, you might find it useful to see how `lazy()` deals with globals (which is done with help of the '[globals]' package).

_Footnote_: (*) The task of identifying globals is a challenging problem and with concurrent/parallel evaluation there will always be corner cases that will not work as intended (and figuring out why can sometimes be tricky, even when troubleshooting using lazy futures).  The purpose of the '[globals]' package is to try to standardize how globals are identified into one or a small number of robust strategies.  Until such a standard has been identified and implemented, the 'future' package will not attempt to provide other types of futures with an automatic service for dealing with globals (except for lazy futures).  This may change in the future (yes, this pun was also intended).


## Demos
To see another illustration how the lazy and eager evaluations of futures differ, run the Mandelbrot demo of this package.  First try with the eager evaluation,
```r
library(future)
plan(eager)
demo("mandelbrot", package="future", ask=FALSE)
```
which closely immitates how the script would run if futures were not used.  Then try the same using lazy evaluation,
```r
plan(lazy)
demo("mandelbrot", package="future", ask=FALSE)
```
and see if you can notice the difference in how and when statements are evaluated.



## Contributing
The goal of this package is to provide a standardized and unified API for using futures in R.  What you are seeing right now is an early but sincere attempt to achieve this goal.  I am open to all types of feedback and I welcome contributions and collaborations of any kind.


[BatchJobs]: http://cran.r-project.org/package=BatchJobs
[listenv]: http://cran.r-project.org/package=listenv
[globals]: http://cran.r-project.org/package=globals
[async]: https://github.com/HenrikBengtsson/async/


## Installation
R package future is only available via [GitHub](https://github.com/HenrikBengtsson/future) and can be installed in R as:
```r
source('http://callr.org/install#HenrikBengtsson/future')
```


## Software status

| Resource:     | GitHub        | Travis CI     | Appveyor         |
| ------------- | ------------------- | ------------- | ---------------- |
| _Platforms:_  | _Multiple_          | _Linux_       | _Windows_        |
| R CMD check   |  | <a href="https://travis-ci.org/HenrikBengtsson/future"><img src="https://travis-ci.org/HenrikBengtsson/future.svg" alt="Build status"></a> | <a href="https://ci.appveyor.com/project/HenrikBengtsson/future"><img src="https://ci.appveyor.com/api/projects/status/github/HenrikBengtsson/future?svg=true" alt="Build status"></a> |
| Test coverage |                     | <a href="https://coveralls.io/r/HenrikBengtsson/future"><img src="https://coveralls.io/repos/HenrikBengtsson/future/badge.svg?branch=develop" alt="Coverage Status"/></a>   |                  |
