# future: A Future API for R

## Introduction
The purpose of the [future] package is to provide a very simple and uniform way of evaluting R expressions asynchroneously using various resources available to the user.

In programming, a _future_ is an abstraction for a _value_ that may be available at some point in the future.  The state of a future can either be _unresolved_ or _resolved_.  As soon as it is resolved, the value is available instantaneously.  If the value is queried while the future is still unresolved, the current process is _blocked_ until the future is resolved.  It is possible to check whether a future is resolved or not without blocking.  Exactly how and when futures are resolved depends on what strategy is used to evaluate them.  For instance, a future can be resolved using a "lazy" strategy, which means it is resolved only when the value is requested.  Another approach is an "eager" strategy, which means that it starts to resolve the future as soon as it is created.  Yet other strategies may be to resolve futures asynchronously, for instance, by evaluating expressions concurrently on a compute cluster.

Here is an example illustrating how the basics of futures work.  First, consider the following code snippet that uses plain R code:
```r
> v <- {
+   cat("Resolving...\n")
+   3.14
+ }
Resolving...
> v
[1] 3.14
```
It works by assigning the value of an expression to variable `v` and we then print the value of `v`.  Moreover, when the expression for `v` is evaluated we also print a message.

Next, here is the same code snippet modified to use futures instead:
```r
> library("future")
> v %<-% {
+   cat("Resolving...\n")
+   3.14
+ }
Resolving...
> v
[1] 3.14
```
The difference is how `v` is constructed; with plain R we use `<-` whereas with futures we use `%<-%`.

So why are futures useful?  Because we can choose to evaluate the future expression in, for instance, a separate R process by a simple switch of settings:
```r
> library("future")
> plan(multiprocess)
> v %<-% {
+   cat("Resolving...\n")
+   3.14
+ }
> v
[1] 3.14
```
With asynchronous futures the current/main R process is _not_ block and available for further processing while the future is being resolved in a separate process in the background.  In other words, futures provide a simply but yet powerful construct for parallel processing in R.


Now, if you cannot be bothered to read all the nitty-gritty details about futures, but just want to try them out, then skip to the end to play with the Mandelbrot demo using both parallel and non-parallel evaluation.



## Implicit or Explicit Futures

Futures can be created either _implicitly_ or _explicitly_.  In the introductory example above we used _implicit futures_ created via the `v %<-% { expr }` construct.  An alternative is _explicit futures_ using the `f <- future({ expr })` and `v <- value(f)` constructs.  For example, our future example could also be written as:
```r
> library("future")
> f <- future({
+   cat("Resolving...\n")
+   3.14
+ })
> v <- value(f)
Resolving...
> v
[1] 3.14
```

Either style of future construct works equally(*) well.  The implicitly style is most similar to how regular R code is written.  In principal all you have to do is to replace `<-` with a `%<-%` to turn the assigment into a future assignment.  On the other hand, this simplicity can also be decieving, particularly when asynchronous futures are being used.  In contrast, the explicit style makes it much more clear that futures are being used, which lowers the risk for mistakes and better communicates the design to others reading your code.

(*) There are cases where `%<-%` cannot be used without some modifications.  We will return to this in Section 'Constraints using Implicit Futures' near the end of this document.



To summarize, for explicit futures, we use:

* `f <- future({ expr })` - creates a future
* `v <- value(f)` - gets the value of the future (blocks if not yet resolved)

For implicit futures, we use:

* `v %<-% { expr }` - creates a future and a promise to its value

To keep it simple, we will use the implicit style in the rest of this document.



## Controlling How Futures are Resolved
The future package implements the following types of futures:

| Name            | OSes        | Description
|:----------------|:------------|:-----------------------------------------------------
| _synchronous:_  |             | _non-parallel:_
| eager           | all         |
| lazy            | all         | lazy evaluation - only happens iff value is requested
| transparent     | all         | for debugging (eager w/ early signalling and w/out local)
| _asynchronous:_ |             | _parallel_:
| multiprocess    | all         | multicore iff supported, otherwise multisession
| multisession    | all         | background R sessions (on current machine)
| multicore       | not Windows | forked R processes (on current machine)
| cluster         | all         | external R sessions on current and/or remote machines

The future package is designed such that support for additional strategies can be implemented as well.  For instance, the future.BatchJobs package (to be published) provides futures for all types of _cluster functions_ ("backends") that the [BatchJobs] package supports.  Specifically, futures for evaluating R expressions via job schedulers such as Slurm, TORQUE/PBS, Oracle/Sun Grid Engine (SGE) and Load Sharing Facility (LSF), will soon be available.

By default, future expression are evaluated synchronously (in the current R session) immediately.  This evaluation strategy is referred to as "eager" and we refer to futures using this strategy as "eager futures".  When can explicitly set this strategy using `plan(eager)`.  In this section we will go through each of these strategies and discuss what they have in common and how they differ.


### Coherent Behavior Across Futures
Before going through each of the different future strategies, it could helpful if we talk about some of the design objectives of the future package and the Future API it defines.  When programming with futures, it should not really matter what future strategy will be used when running the code.  This is because we cannot really know what resources the user have access to so the choice of evaluation strategy should be in the hand of the user and not the developer.  In other words, the code should not make any assumptions on type of futures, e.g. synchronous or asynchronous.

One of the designs of the Future API was to encapsulate any differences such that all types of futures will appear to work the same.  This despite the expression may be evaluate locally in the current R process or across the world in a remote R session.  Another obvious advantage of having a consist API and behavior among different types of futures is that it helps prototyping.  Typically one would set up a script using eager evaluation and when fully tested one may turn on asynchronous processing.

Because of this, the defaults of of the different strategies are such that the results and side effects of evaluating a future expression are as similar as possible.  More specifically, the following is true for all futures:

* All _evaluation is done in a local environment_ (e.g. `local({ expr })`) such assignments do not affect the calling environment.  This is natural when evaluating in an external R process, but is also enforced when evaluating in the current R session.

* When a future is constructed, _global variables are identified and validated_.  For lazy evaluation, the globals are also "frozen" (cloned to a local environment) until needed.  For asynchronous evaluation, they are also exported to the R process/session that will be evaluating the future expression.  Regardless of strategy, globals that cannot be located will cause an informative error.  If too large globals (according an option) are about to be exported, an informative error is also generated.

* Future _expressions are only evaluated once_.  As soon as the value has been collected it will be available for all succeeding requests.

Here is an example illustrating that all assignments are done to a local environment:
```r
> plan(eager)
> a <- 1
> x %<-% {
+     a <- 2
+     2 * a
+ }
> x
[1] 4
> a
[1] 1
```

And here is an example illustrating that globals are validated already when the future is created:
```r
> rm(b)
> x %<-% { 2 * b }
Error in globalsOf(expr, envir = envir, substitute = FALSE, tweak = tweak,  :
  Identified a global object via static code inspection ({; 2 * b; }), but
failed to locate the corresponding object in the relevant environments: 'b'
```
We will return to global variables and functions in Section 'Globals' near the end of this document.

Now we are ready to explore the different future strategies.


### Synchronous Futures

#### Eager Futures
Eager futures are the default unless otherwise specified.  The are designed to behave as similar to regular R evaluation as possible while still fullfilling the Future API and behaviors.  Here is an example illustrating their properties:
```r
> plan(eager)
> pid <- Sys.getpid()
> pid
[1] 7712
> a %<-% {
+     pid <- Sys.getpid()
+     cat("Resolving 'a' ...\n")
+     3.14
+ }
Resolving 'a' ...
> b %<-% {
+     rm(pid)
+     cat("Resolving 'b' ...\n")
+     Sys.getpid()
+ }
Resolving 'b' ...
> c %<-% {
+     cat("Resolving 'c' ...\n")
+     2 * a
+ }
Resolving 'c' ...
> b
[1] 7712
> c
[1] 6.28
> a
[1] 3.14
> pid
[1] 7712
```
Since eager evaluation is taking place, each of the three futures is resolved instantaneously in the moment it is created.  Note also how `pid`, which is the process ID of the current process, is neither overwritten nor removed.  Since synchronous processing is used, future `b` is evaluated in the current (the calling) process, which is why the value of `b` and `pid` are the same.


#### Lazy Futures
A lazy future evaluates its expression only if its value is queried.  It will occur if the future is checked for being resolved or not.  Here is the above example when using lazy evaluation:
```r
> plan(lazy)
> pid <- Sys.getpid()
> pid
[1] 7712
> a %<-% {
+     pid <- Sys.getpid()
+     cat("Resolving 'a' ...\n")
+     3.14
+ }
> b %<-% {
+     rm(pid)
+     cat("Resolving 'b' ...\n")
+     Sys.getpid()
+ }
> c %<-% {
+     cat("Resolving 'c' ...\n")
+     2 * a
+ }
Resolving 'a' ...
> b
Resolving 'b' ...
[1] 7712
> c
Resolving 'c' ...
[1] 6.28
> a
[1] 3.14
> pid
[1] 7712
```
As previously, variable `pid` is unaffected because all evaluation is done in a local environment.  More interestingly, future `a` is no longer evaluated in the moment it is created, but instead when it is needed the first time, which happens when future `c` is created.  This is because `a` is identified as a global variable that needs to be captured ("frozen" to `a == 3.14`) in order to set up future `c`.  Later when `c` (the value of future `c`) is queried, `a` has already been resolved and only the expression for future `c` is evaluated and `6.14` is obtained.  Moreover, future `b` is just like `a` evaluated only when it is needed the first time, i.e. when `b` is printed.  As for eager evaluation, lazy evaluation is also synchronous which is why `b` and `pid` have the same value.  Finally, notice also how `a` is not re-evaluated when we query the value again.  Actually, with implicit futures, variables `a`, `b` and `c` are all regular values as soon as their futures have been resolved.

_Comment_: Lazy evaluation is already supported by R itself.  Arguments are passed to functions using lazy evaluation.  It is also possible to assign variables  using lazy evaluation using `delayedAssign()`, but contrary to lazy futures this function does not freeze globals.  For more information, see `help("delayedAssign", package="base")`.






### Asynchronous Futures

#### Multisession Futures
Next we will turn to asynchronous futures.  We start with multisession futures because they supported by all operating systems.  A multisession future is evaluated in a background R session running on the same machine as the calling R process.  Here is our example with multisession evaluation:
```r
> plan(multisession)
> pid <- Sys.getpid()
> pid
[1] 7712
> a %<-% {
+     pid <- Sys.getpid()
+     cat("Resolving 'a' ...\n")
+     3.14
+ }
> b %<-% {
+     rm(pid)
+     cat("Resolving 'b' ...\n")
+     Sys.getpid()
+ }
> c %<-% {
+     cat("Resolving 'c' ...\n")
+     2 * a
+ }
> b
[1] 7744
> c
[1] 6.28
> a
[1] 3.14
> pid
[1] 7712
```
The first thing we observe is that the values of `a`, `c` and `pid` are the same are previously.  However, we noticed that `b` is different from before.  This is because the future `b` is evaluated in a different R process and therefore it returns a different process ID.  Another difference is that the messages, generated by `cat()`, are no longer displayed.  This is because they are outputted to the background sessions and not the calling session.

When multisession evaluation is used, the package launches a set of R sessions in the background that will serve multisession futures by evaluating their expressions as they are created.  If all background sessions are busy evaluating other future expressions, the creation of the next multisession future is _blocked_ until a background session becomes available again.  The total number of background processes launched is decided by the value of `availableCores()`, e.g.
```r
> availableCores()
mc.cores+1 
         3 
```
This particular result tells us that option `mc.cores` was set such that we are allowed to use in total 3 processes including the main process.  In other words, with these settings there will be 2 background processes serving the multisession futures.  The `availableCores()` is also agile to different options and system environment variables.  For instance, if compute cluster schedulers are used (e.g. TORQUE/PBS and Slurm), they set specific environment variable specifying the number of cores that was alloted to any given job; `availableCores()` acknowledges these as well.  If nothing else is specified, all available cores on the machine will be utilized, cf. `parallel::detectCores()`.  For more details, please see `help("availableCores", package="future")`.


#### Multicore Futures
On operating systems where R supports _forking_ of processes, which is basically all operating system except Windows, an alternative to spawning R sessions in the background is to fork the existing R process.  Forking an R process is considered faster working with a separate R session running in the background.  One reason is that the overhead of exporting large globals to the background session can be greater than when forking is used.  Other than this, the behavior of using multicore evaluation is very similar to using multisession evaluation.
To use multicore futures, we specify:
```r
plan(multicore)
```
Except for different process IDs, the output of our example using multicore evaluation will be the same as that for multisession evaluation.   In both cases the evaluation is done on the local machine.  Just like for multisession futures, the maximum number of parallel processes running will be decided by `availableCores()`.



#### Multiprocess Futures
Sometimes we do not know whether multicore futures are supported or not, but it might still be that we would like to write platform-independent scripts or instructions that works everywhere.  In such cases we can specify that we want to use "multiprocess" futures as in:
```r
plan(multiprocess)
```
A multiprocess future is not a formal class of futures by itself, but rather a convenient alias for either of the two.  When this is specified, multisession evaluation will be used unless multicore evaluation is supported.


#### Cluster Futures
Cluster futures evaulates expression on an ad-hoc cluster that was set up via `parallel::makeCluster()`.  For instance, assume you have access to three nodes `n1`, `n2` and `n3`, you can use these for asynchronous evaluation as:
```r
> hosts <- c("n1", "n2", "n3")
> cl <- parallel::makeCluster(hosts)
> plan(cluster, cluster = cl)
> pid <- Sys.getpid()
> pid
[1] 7712
> a %<-% {
+     pid <- Sys.getpid()
+     cat("Resolving 'a' ...\n")
+     3.14
+ }
> b %<-% {
+     rm(pid)
+     cat("Resolving 'b' ...\n")
+     Sys.getpid()
+ }
> c %<-% {
+     cat("Resolving 'c' ...\n")
+     2 * a
+ }
> b
[1] 7896
> c
[1] 6.28
> a
[1] 3.14
> pid
[1] 7712
```
Just as for the other asynchronous evaluation strategies, the output from `cat()` is not displayed on the current/calling machine.  By the way, it is considered good style to shutdown the cluster when it is no longer needed, that is, calling `parallel::stopCluster(cl)`.  However, it will shut itself done if the main process is terminated.  For more information on how to setup and manage such clusters, see `help("makeCluster", package="parallel")`.

Note that with proper configuration and automatic authentication setup (e.g. SSH key pairs), there is nothing preventing us from using the same approach for using a cluster of remote machines.



### Different Strategies for Different Futures
Sometimes one may want to use an alternative evaluation strategy for a specific future.  Although one can use `old <- plan(new)` and afterward `plan(old)` to temporarily switch strategies, a simpler approach is to use the `%plan%` operator, e.g.
```r
> plan(eager)
> pid <- Sys.getpid()
> pid
[1] 7712
> a %<-% {
+     Sys.getpid()
+ }
> b %<-% {
+     Sys.getpid()
+ } %plan% multiprocess
> c %<-% {
+     Sys.getpid()
+ } %plan% multiprocess
> a
[1] 7712
> b
[1] 3272
> c
[1] 7744
```
As seen by the different process IDs, future `a` is evaluated eagerly using the same process as the calling environment whereas the other two are evaluated using multiprocess futures.

However, using `%plan%` has the drawback of hard coding the evaluation strategy.  Doing so, may prevents some users from using your script or package, because they do not have the sufficient resources.  It may also prevent users with large amount of resources from utilizing those, because you assumed a less-powerful set of hardware.  Because of this, we recommend against the use of `%plan%` other than for interactive protyping.


### Nested Futures and Evaluation Topologies
This far we have discussed what can be referred to as "flat topology" of futures, that is, all futures are created in and assigned to the same environment.  However, there is nothing preventing us from using a "nested topology" of futures, where one set of futures may in turn create another set of futures internally and so on.

For instance, here is an example of two "top" futures (`a` and `b`) that uses multiprocess evaluation and where the second future (`b`) in turn uses two internal futures:
```r
> plan(multiprocess)
> pid <- Sys.getpid()
> a %<-% {
+     cat("Resolving 'a' ...\n")
+     Sys.getpid()
+ }
> b %<-% {
+     cat("Resolving 'b' ...\n")
+     b.1 %<-% {
+         cat("Resolving 'b.1' ...\n")
+         Sys.getpid()
+     }
+     b.2 %<-% {
+         cat("Resolving 'b.2' ...\n")
+         Sys.getpid()
+     }
+     c(b.pid = Sys.getpid(), b.1.pid = b.1, b.2.pid = b.2)
+ }
> pid
[1] 7712
> a
[1] 3272
> b
  b.pid b.1.pid b.2.pid 
   7744    7744    7744 
```
By inspection the process IDs, we see that there are in total three different processes involved.  There is the main R process (pid=7712), and there are the two processes used by `a` (pid=3272) and `b` (pid=7744).  However, the two futures (`b.1` and `b.2`) that is nested by `b` are evaluated by the same R process as `b`.  The is because nested futures use eager evaluation unless otherwise specified.  There are a few reasons for this, but the main reason is that it prevents spawning of a large number of background processes by mistake, e.g. via recursive calls.

To specify a different type of _evaluation topology_ than the first level of futures being resolved by multiprocess evaluation and the second level by eager evaluation, we can provide a list of evaluation strategies to `plan()`.  First, the same evaluation strategies as above can be explicitly specified as:
```r
plan(list(multiprocess, eager))
```
We would actually get the same behavior if we try with multiple levels of multiprocess evaluations;
```r
> plan(list(multiprocess, multiprocess))
[...]
> pid
[1] 7712
> a
[1] 3272
> b
  b.pid b.1.pid b.2.pid 
   7744    7744    7744 
```
The reason for this also here to protect against launching more processes than what the machine can support.  Internally, this is done by setting `mc.cores` to zero ([sic!](https://github.com/HenrikBengtsson/Wishlist-for-R/issues/7)) such that no _additional_ parallel processes can be launched.  This is the case for both multisession and multicore evaluation.

Continuing, if we start off by eager evaluation and then use multiprocess evaluation for any nested futures, we get:
```r
> plan(list(eager, multiprocess))
[...]
Resolving 'a' ...
Resolving 'b' ...
> pid
[1] 7712
> a
[1] 7712
> b
  b.pid b.1.pid b.2.pid 
   7712    3272    7744 
```
which clearly show that `a` and `b` are resolved in the calling process (pid 7712) whereas the two nested futures (`b.1` and `b.2`) are resolved in two separate R processes (pids 3272 and 7744).

Having said this, it is indeed possible to use nested multiprocess evaluation strategies, if we explicitly specify (read _force_) the number of cores available at each level.  In order to do this we need to "tweak" the default settings, which can be done as follows:
```r
> plan(list(tweak(multiprocess, maxCores = 3), tweak(multiprocess, 
+     maxCores = 3)))
[...]
> pid
[1] 7712
> a
[1] 3272
> b
  b.pid b.1.pid b.2.pid 
   7744    8028    4680 
```
First, we see that both `a` and `b` are resolved in different processes (pids 3272 and 7744) than the calling process (pid 7712).  Second, the two nested futures (`b.1` and `b.2`) are resolved in yet two other R processes (pids 8028 and 4680).

To clarify, when we setup the two levels of multiprocess evaluation, we specified that in total 3 processes may be used at each level.  We choose three parallel processes and not just two, because one is always consumed by the calling process leaving two to be used for the asynchronous futures.  This is why we see that `pid`, `a` and `b` are all resolved in processes.  If we had allowed only two cores at the top level, `a` and `b` would have been resolved by the same background process.  The same applies for the second level of futures.  This bring us to another point.  When we use asynchronous futures, there is nothing per se that prevents us from using the main process to keep doing computations while checking in ot the futures now and then to see if they are resolved.

For more details on working with nested futures and different evaluation strategies at each level, see Vignette '[Futures in R: Future Topologies]'.


### Checking A Future without Blocking
It is possible to check whether a future has been resolved or not without blocking.  This can be done using the `resolved(f)` function, which takes an explicit future `f` as input.  If we work with implit futures (as in all the examples above), we can use the `f <- futureOf(a)` function to retrieve the implicit future from an explicit one.  For example,
```r
> plan(multiprocess)
> a %<-% {
+     cat("Resolving 'a' ...")
+     Sys.sleep(2)
+     cat("done\n")
+     Sys.getpid()
+ }
> cat("Waiting for 'a' to be resolved ...\n")
Waiting for 'a' to be resolved ...
> fa <- futureOf(a)
> count <- 1
> while (!resolved(fa)) {
+     cat(count, "\n")
+     Sys.sleep(0.2)
+     count <- count + 1
+ }
1 
2 
3 
> cat("Waiting for 'a' to be resolved ... DONE\n")
Waiting for 'a' to be resolved ... DONE
> a
[1] 3272
```

It is possible to nest futures in multiple levels and each of the nested futures may be resolved using a different strategy

When using asynchronous (multicore, multisession and cluster) futures, recursive asynchronous evaluation done by mistake is protected against by forcing eager futures and option `mc.cores` to zero (number of _additional_ cores available for processing in addition to the main process) to lower the risk for other multicore mechanisms to spawn off additional cores.
In order to use other types of future strategies for the nested layers of futures, one may give a list of strategies to `plan()`.  For more details, see Vignette '[Futures in R: Future Topologies]'.


## Failed Futures
Sometimes the future is not what you expected.  If an error occurs while evaluating a future, the error is propagated and thrown as an error in the calling environment _when the future value is requested_.  For example,
```r
> plan(lazy)
> a %<-% {
+     cat("Resolving 'a' ...")
+     stop("Whoops!")
+     42
+ }
> cat("Everything is still ok although we have created a future that will fail.\n")
Everything is still ok although we have created a future that will fail.
Resolving 'a' ...> a
Resolving 'a' ...
Error in eval(expr, envir, enclos) : Whoops!
```
The error is thrown each time the value is requested, that is, if we try to get the value again will generate the same error:
```r
> a
Error in eval(expr, envir, enclos) : Whoops!
In addition: Warning message:
restarting interrupted promise evaluation
```
To see the list of calls (evaluated expressions) that lead up to the error, we can use the `backtrace()` function(*) on the future, i.e.
```r
> backtrace(a)
[[1]]
eval(quote({
    cat("Resolving 'a' ...")
    stop("Whoops!")
    42
}), new.env())
[[2]]
eval(expr, envir, enclos)
[[3]]
stop("Whoops!")
```
(*) The regular `traceback()` available in R does not give useful information.


## Globals
Whenever an R expression is to be evaluated asynchronously (in parallel) or via lazy evaluation, global objects need to be identified and passed to the evaluator.  They need to be passed exactly as they were at the time the future was created, because, for a lazy future, globals may otherwise change between when it is created and when it is resolved.  For asynchronously, the reason is that globals need to be exported to the process that evaluates the future.

The future package tries to automate the identification of globals in future expressions.  It does this with help from the [globals] package.  If a global variable is identified, it is captured and made available to the evaluating process.  If it identifies a symbol that it believes is a global object, but it fails to locate it in the calling environment (or any the environment accessible from that one), an error is thrown immediately.  This minimizing the risk for runtime errors occurring later (sometimes much later) and in a different process (possible in a remote R session).  For instance,
```r
> rm(a)
> x <- 5.0
> y %<-% { a * x }
Error in globalsOf(expr, envir = envir, substitute = FALSE, tweak = tweak,  :
  Identified a global object via static code inspection ({; a * x; }), but
failed to locate the corresponding object in the relevant environments: 'a'

> a <- 1.8
> y %<-% { a * x }
> y
[1] 9
```
Moreover, if a global is defined in a packages, for instance a function, then that global is not exported but instead it is made sure that the corresponding package is attached when the future is evaluated.  This not only better reflects the setup of the main R session, but it also minimizes the need for exporting globals, which saves time and bandwidth, especially when using remote compute nodes.

As mentioned previously, for consistency across evaluation strategies, all types of futures validate globals upon creation.  This is also true for cases where it would not be necessary, e.g. for eager evaluation of multicore evaluation (which forks the calling process with all of its objects "as-is").  However, in order to make it as easy as possible to switch between strategies without being surprised by slightly different behaviors, the Future API is designed to check for globals the same way regardless of strategy.

Having said this, it is possible to disable validation of globals by setting `globals=FALSE`.  This could make sense if one know for sure that either eager or multicore futures will be used.  This argument can be tweaked as `plan(tweak(eager, globals=FALSE)` and `plan(tweak(multicore, globals=FALSE)`.  However, it is strongly recommended not to do this.  Instead, as a best practice, it always  good if the code/script works with any type of futures.

Finally, it is a challenging problem to identify globals from static code inspection.  There will always be corner cases of globals that either fails to be identified by static code inspection or that are incorrectly identified as global variables.  Vignette '[Futures in R: Common Issues with Solutions]' provides examples of common cases and explains how to avoid them as well as how to help the package to identify globals or to ignore falsely identified globals.



## Constraints  using Implicit Futures

There is one limitation with implicit futures that does not exist for explicit ones.  Because an explicit future is just like any other object in R it can be assigned anywhere/to anything.  For instance, we can create several of them in a loop and assign them to a list, e.g.
```r
> plan(multiprocess)
> f <- list()
> for (ii in 1:3) {
+     f[[ii]] <- future({
+         Sys.getpid()
+     })
+ }
> v <- lapply(f, FUN = value)
> str(v)
List of 3
 $ : int 3272
 $ : int 7744
 $ : int 3272
```
This is _not_ possible to do when using implicit futures.  This is because the `%<-%` assignment operator _cannot_ be used in all cases where the regular `<-` assignment operator can be used.  It can only be used to assign future values to _environments_ (including the calling environment) much like how `assign(name, value, envir)` works.  However, we can assign implicit futures to environments using _named indices_, e.g.
```r
> plan(multiprocess)
> v <- new.env()
> for (name in c("a", "b", "c")) {
+     v[[name]] %<-% {
+         Sys.getpid()
+     }
+ }
> v <- as.list(v)
> str(v)
List of 3
 $ a: int 3272
 $ b: int 7744
 $ c: int 3272
```
Here `as.list(v)` blocks until all futures in the environment `v` have been resolved.  Then their values are collected and returned as a regular list.

If _numeric indices_ are required, then _list environments_ can be used.  List environments, which are implemented by the [listenv] package, are regular environments with customized subsetting operators making it possible to index them much like how lists can be indexed.  By using list environments where we otherwise would use lists, we can also assign implicit futures to a list-like objects using numeric indices.  For example,
```r
> library("listenv")
> plan(multiprocess)
> v <- listenv()
> for (ii in 1:3) {
+     v[[ii]] %<-% {
+         Sys.getpid()
+     }
+ }
> v <- as.list(v)
> str(v)
List of 3
 $ : int 3272
 $ : int 7744
 $ : int 3272
```
As previously, `as.list(v)` blocks until all futures.



## Demos
To see a live illustration how different types of futures are evaluated, run the Mandelbrot demo of this package.  First try with the eager evaluation,
```r
library("future")
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
It's always good to call `parallel::stopCluster(cl)` when you're done with this cluster, but if you forget it will be done automatically when you quit R.



## Contributing
The goal of this package is to provide a standardized and unified API for using futures in R.  What you are seeing right now is an early but sincere attempt to achieve this goal.  If you have comments or ideas on how to improve the 'future' package, I would love to hear about them.  The preferred way to get in touch is via the [GitHub repository](https://github.com/HenrikBengtsson/future/), where you also find the latest source code.  I am also open to contributions and collaborations of any kind.


[BatchJobs]: http://cran.r-project.org/package=BatchJobs
[future]: http://cran.r-project.org/package=future
[globals]: http://cran.r-project.org/package=globals
[listenv]: http://cran.r-project.org/package=listenv
[Futures in R: Common Issues with Solutions]: future-2-issues.html
[Futures in R: Future Topologies]: future-3-topologies.html

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
