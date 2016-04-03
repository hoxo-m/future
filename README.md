# future: A Future API for R

## Introduction
In programming, a _future_ is an abstraction for a _value_ that may be available at some point in the future.  The state of a future can either be _unresolved_ or _resolved_.  As soon as it is resolved, the value is available instantaneously.  If the value is queried while the future is still unresolved, the current process is _blocked_ until the future is resolved.  Exactly how and when futures are resolved depends on what strategy is used to evaluate them.  For instance, a future can be resolved using a "lazy" strategy, which means it is resolved only when the value is requested.  Another approach is an "eager" strategy, which means that it starts to resolve the future as soon as it is created.  Yet other strategies may be to resolve futures asynchronously, for instance, by evaluating expressions concurrently on a compute cluster.

### Futures in R

The purpose of the 'future' package is to define and provide a minimalistic Future API for R.  The package itself provides two _synchronous_ mechanisms for "lazy" and "eager" futures, and a three _asynchronous_ ones for "multicore", "multisession" and "cluster" futures.  Further strategies will be implemented by other packages extending the 'future' package.  For instance, the 'async' package resolves futures _asynchronously_ via any of the many backends that the '[BatchJobs]' framework provides, e.g. distributed on a compute cluster via a job queue/scheduler.  The lazy and the eager futures, which are both synchronous (blocks the main process while being resolved), are useful as a last test before using asynchronous futures.

Here is an example illustrating how the basics of futures work:

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
Note how the future is resolved as soon as we create it using `future()`.  This is because the default strategy for resolving futures is to evaluate them in an "eager" and synchronous manner, which emulates how R itself typically evaluates expressions, cf.
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

For troubleshooting, there is also a _transparent_ future, which can be specified as `plan(transparent)`.  A transparent future is technically an eager future with instant signaling of conditions (including errors and warnings) and where evaluation, and therefore also assignments, take place in the calling environment.  Transparent futures are particularly useful for troubleshooting errors.


### Promises of successful futures
An important part of a future is the fact that, although we do not necessarily control _when_ a future is resolved, we do have a "promise" that it _will_ be resolved (at least if its value is requested).  In other words, if we ask for the value of a future, we are guaranteed that the expression of the future will be evaluated and its value will be returned to us (or an error will be generated if the evaluation caused an error).  An alternative to a future-value pair of function calls is to use the `%<-%` infix assignment operator (also provided by the 'future' package).  For example,

```r
> plan(lazy)
> v %<-% {
+   message("Resolving...")
+   3.14
+ }
> v
Resolving...
[1] 3.14
```

This works by (i) creating a future and (ii) assigning its value to variable `v` as a _promise_.  Specifically, the expression/value assigned to variable `v` is promised to be evaluated (no later than) when it is requested.  Promises are built-in constructs of R (see `help(delayedAssign)`).
To get the future of a future variable, use the `futureOf()` function, e.g. `f <- futureOf(v)`.


### Eager, lazy and parallel futures
You are responsible for your own futures and how you choose to resolve them may differ depending on your needs and your resources.  The 'future' package provides two _synchronous_ futures, the "lazy" and "eager" futures, implemented by functions `lazy()` and `eager()`.
It also provides different types of _asynchronous_ futures, e.g. the "multicore" and the "multisession" futures, implemented by functions `multicore()` and  `multisession()`.  The multicore future is available on systems where R supports process forking, that is, on Unix-like operating systems, but not on Windows.  On non-supported systems, multicore futures automatically become eager futures.
The multisession future is available on all systems, including Windows, and instead of forking the current R process, it launches a set of R sessions in the background on which the multisession futures are evaluated.  Both multicore and multisession evaluation is agile to the number of cores available to the R session running, which includes acknowledging the `mc.cores` options among other settings.  For details, see `help("availableCores", package="future")`.
To use multicore futures where supported and otherwise multisession ones, one can use the more general _multiprocess_ futures, i.e. `plan(multiprocess)`.
There is also a more generic "cluster" future as implemented by `cluster()`, which makes it possible to use any type of cluster created by `parallel::makeCluster()`.

Since an asynchronous strategy is more likely to be used in practice, the built-in eager and lazy mechanisms try to emulate those as far as possible while at the same time evaluating them in a synchronous way.  For example, the default for all types of futures is that the expression is evaluated in _a local environment_ (cf. `help("local")`), which means that any assignments are done to local variables only - such that the environment of the main/calling process is unaffected.  Here is an example:

```r
> a <- 2.71
> x %<-% { a <- 3.14 }
> x
[1] 3.14
> a
[1] 2.71
```
This shows that `a` in the calling environment is unaffected by the expression evaluated by the future.


### Different strategies for different futures
Sometimes one may want to use an alternative evaluation strategy for a specific future.  Although one can use `old <- plan(new)` and afterward `plan(old)` to temporarily switch strategies, a simpler approach is to use the `%plan%` operator, e.g.
```r
> plan(eager)
> a <- 0
> x %<-% { 3.14 }
> y %<-% { a <- 2.71 } %plan% lazy(local=FALSE, globals=FALSE)
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
It is possible to nest futures in multiple levels and each of the nested futures may be resolved using a different strategy, e.g.
```r
> plan(lazy)
> c %<-% {
+   message("Resolving 'c'")
+   a %<-% {
+     message("Resolving 'a'")
+     3
+   } %plan% eager
+   b %<-% {
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

When using asynchronous (multicore, multisession and cluster) futures, recursive asynchronous evaluation done by mistake is protected against by forcing option `mc.cores` to zero (number of _additional_ cores available for processing in addition to the main process) to lower the risk for other multicore mechanisms to spawn off additional cores.  If nested asynchronous futures are truly wanted, it is possible to override this by setting `mc.cores` and/or use another type of future in nested calls by explicitly doing so as part of the future expression.


## Assigning futures to environments and list environments
The `%<-%` assignment operator _cannot_ be used in all cases where the regular `<-` assignment operator can be used.  For instance, it is _not_ possible to assign future values to a _list_;

```r
> x <- list()
> x$a %<-% { 2.71 }
Error: Subsetting can not be done on a 'list'; only to an environment: 'x$a'
```

This is because _promises_ themselves cannot be assigned to lists.  More precisely, the limitation of future assignments are the same as those for assignments via the `assign()` function, which means you can only assign _future values_ to environments (defaulting to the current environment) but nothing else, i.e. not to elements of a vector, matrix, list or a data.frame and so on.  To assign a future value to an environment, do:

```r
> env <- new.env()
> env$a %<-% { 1 }
> env[["b"]] %<-% { 2 }
> name <- "c"
> env[[name]] %<-% { 3 }
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
+   x[[ii]] %<-% { rnorm(ii) }
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
+   message("Resolving...")
+   stop("Whoops!")
+   42
+ })
> value(f)
Resolving...
Error in eval(expr, envir, enclos) : Whoops!
```
The error is thrown each time the value is requested, that is, trying to get the value again will generate the same error:
```r
> value(f)
Error in eval(expr, envir, enclos) : Whoops!
```
Note how the future expression is only evaluated once although the error itself is re-thrown each time the value is required subsequently.

Exception handling of future assignments via `%<-%` works analogously, e.g.
```r
> plan(lazy)
> x %<-% {
+   message("Resolving...")
+   stop("Whoops!")
+   42
+ }
> y <- 3.14
> y
[1] 3.14
> x
Resolving...
Error in eval(expr, envir, enclos) : Whoops!
> x
Error in eval(expr, envir, enclos) : Whoops!
In addition: Warning message:
restarting interrupted promise evaluation
```


## Globals
Whenever an R expression is to be evaluated asynchronously (in parallel) or via lazy evaluation, global objects need to be identified and passed to the evaluator.  They need to be passed exactly as they were at the time the future was created, because, for a lazy future, globals may otherwise change between when it is created and when it is resolved.

The future package tries to automate the identification of globals in future expressions.  It does so with help of the [globals] package.  If a global variable is identified, it is captured and made available to the evaluator of the future, e.g. it is exported to the work environment of an R session running in the background.  If it identifies a symbol that it believes is a global object in the future expression, but it fails to locate it in the work environment, an error is thrown immediately (minimizing the risk for runtime errors occurring much later).  For instance,
```r
> x <- 5.0
> y %<-% { a * x }
Error in globalsOf(expr, envir = envir, substitute = FALSE, tweak = tweak,  :
Identified a global by static code inspection, but failed to locate the
corresponding object in the relevant environments: 'a'
> a <- 1.8
> y %<-% { a * x }
> y
[1] 9
```
Moreover, if a global is defined in a packages, for instance a function, then that global is not exported but instead it is made sure that the corresponding package is attached when the future is evaluated.  This not only reflects the setup of the main R session, but it also minimizes the need for exporting globals, which can save bandwidth and time, especially when using remote compute nodes.

Having said this, it is a challenging problem to identify globals from static code inspection.  There will always be corner cases of globals that either fails to be identified by static code inspection or that are incorrectly identified as global variables.  Vignette '[Futures in R: Common issues with solutions]' provides examples of common cases and explains how to avoid them.
If you suspect that a global variable is not properly identified, it is often helpful for troubleshooting to run the code interactively using synchronous futures, i.e. _eager_ or _lazy_.  If there is an error, it is then possible to use `traceback()` and other debugging tools.

For consistency, all types of futures validates globals upon creation (`globals=TRUE` is the default).  Now, since eager futures are resolved immediately upon creation, any globals will also be resolved at this time and therefore there is actually no need for globals to be identified and validated.  Similarly, because multicore futures fork the main R session when created, globals are automatically "frozen" for each multicore future.  If you prefer not to use the strict validation of globals that the future package does when using these two types of futures, you can disable the check by specifying `plan(eager, globals=FALSE)` and `plan(multicore, globals=FALSE)`, respectively.


## Demos
To see a live illustration how different types of futures are evaluated, run the Mandelbrot demo of this package.  First try with the eager evaluation,
```r
library(future)
plan(eager)
demo("mandelbrot", package="future", ask=FALSE)
```
which closely imitates how the script would run if futures were not used.  Then try the same using lazy evaluation,
```r
plan(lazy)
demo("mandelbrot", package="future", ask=FALSE)
```
and see if you can notice the difference in how and when statements are evaluated.
You may also try multiprocess evaluation, which calculates the different Mandelbrot planes using parallel R processes running in the background.  Try,
```r
plan(multiprocess)
demo("mandelbrot", package="future", ask=FALSE)
```
This will use multicore processing if you are on a system where R supports process forking, otherwise (such as on Windows) it will use multisession processing.

Finally, if you have access to multiple machines you can try to setup a cluster of workers and use them, e.g.
```r
cl <- parallel::makeCluster(c("n2", "n5", "n6", "n6", "n9"))
plan(cluster, cluster=cl)
demo("mandelbrot", package="future", ask=FALSE)
```
Don't forget to call `parallel::stopCluster(cl)` when you're done with this cluster.  If you forget, it will automatically shutdown when you close your R session.



## Contributing
The goal of this package is to provide a standardized and unified API for using futures in R.  What you are seeing right now is an early but sincere attempt to achieve this goal.  If you have comments or ideas on how to improve the 'future' package, I would love to hear about them.  The preferred way to get in touch is via the [GitHub repository](https://github.com/HenrikBengtsson/future/), where you also find the latest source code.  I am also open to contributions and collaborations of any kind.


[BatchJobs]: http://cran.r-project.org/package=BatchJobs
[listenv]: http://cran.r-project.org/package=listenv
[globals]: http://cran.r-project.org/package=globals
[async]: https://github.com/HenrikBengtsson/async/
[Futures in R: Common issues with solutions]: future-issues.html


## Installation
R package future is available on [CRAN](http://cran.r-project.org/package=future) and can be installed in R as:
```r
install.packages('future')
```

### Pre-release version

To install the pre-release version that is available in branch `develop`, use:
```r
source('http://callr.org/install#HenrikBengtsson/future@develop')
```
This will install the package from source.  



## Software status

| Resource:     | CRAN        | Travis CI     | Appveyor         |
| ------------- | ------------------- | ------------- | ---------------- |
| _Platforms:_  | _Multiple_          | _Linux_       | _Windows_        |
| R CMD check   | <a href="http://cran.r-project.org/web/checks/check_results_future.html"><img border="0" src="http://www.r-pkg.org/badges/version/future" alt="CRAN version"></a> | <a href="https://travis-ci.org/HenrikBengtsson/future"><img src="https://travis-ci.org/HenrikBengtsson/future.svg" alt="Build status"></a> | <a href="https://ci.appveyor.com/project/HenrikBengtsson/future"><img src="https://ci.appveyor.com/api/projects/status/github/HenrikBengtsson/future?svg=true" alt="Build status"></a> |
| Test coverage |                     | <a href="https://coveralls.io/r/HenrikBengtsson/future"><img src="https://coveralls.io/repos/HenrikBengtsson/future/badge.svg?branch=develop" alt="Coverage Status"/></a>   |                  |
