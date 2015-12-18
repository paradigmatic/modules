---
title: "Modules in R"
author: "Sebastian Warnholz"
date: "2015-12-18"
output: rmarkdown::html_vignette
vignette: >
  %\VignetteIndexEntry{Modules in R}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}
---

[![Build Status](https://travis-ci.org/wahani/module.png?branch=master)](https://travis-ci.org/wahani/module)
[![codecov.io](https://codecov.io/github/wahani/module/coverage.svg?branch=master)](https://codecov.io/github/wahani/module?branch=master)
[![CRAN](http://www.r-pkg.org/badges/version/module)](http://cran.rstudio.com/package=module)
[![Downloads](http://cranlogs.r-pkg.org/badges/module?color=brightgreen)](http://www.r-pkg.org/pkg/module)

Provides modules as an organizational unit for source code. Modules enforce to be more rigorous when defining dependencies and have a local search path. They can be used as a sub unit within packages or in scripts.

## Installation

From this GitHub:


```r
devtools::install_github("wahani/aoos")
```

# Introduction

The key idea of this package is to provide a unit which is self contained, i.e. 
has it's own scope. The main and most reliable infrastructure for such 
organizational units of source code in the R ecosystem is a package. Compared to
a package modules can be considered ad hoc, but - in the sense of an R-package -
self contained. Furthermore modules consist of one file; in contrast to a
package which can wrap an arbitrary number of files.

There are two use cases. First when you use modules to develop scripts, which is
subject of this section; And then inside of packages where modules act more like
objects, as in object-oriented-programming. Outside of packages modules know
only of the base environment, i.e. within a module the base environment is the
only *package* on the *search path*. Also they are always represented as a list
inside R. Thus they can be treated as bags of functions.

In the following examples you will see the function `module` to define modules.
Typically you do not have to call that function explicitly but instead call
`use` to load a module into your current session.


```r
library(module)
m <- module({
  boringFunction <- function() cat("boring output")
})

m$boringFunction()
```

```
## boring output
```

Since they are isolated from the `.GlobalEnv` the following object `hey` can not
be found:


```r
hey <- "hey"
m <- module({
  isolatedFunction <- function() hey
})
m$isolatedFunction()
```

```
## Error in m$isolatedFunction(): object 'hey' not found
```

## Imports

If you rely on exported objects of packages you can refer to them explicitly
using `::`:


```r
m <- module({
  functionWithDep <- function(x) stats::median(x)
})
m$functionWithDep(1:10)
```

```
## [1] 5.5
```

Or you can use `import` for *attaching* single objects or packages and `use` for
*attaching* or loading a module:


```r
m <- module({
 
  import(stats, median) # make median from package stats available
  
  functionWithDep <- function(x) median(x)

})
m$functionWithDep(1:10)
```

```
## [1] 5.5
```


```r
m <- module({
  
  import(stats)
  
  functionWithDep <- function(x) median(x)

})
m$functionWithDep(1:10)
```

```
## [1] 5.5
```

## Exports

It may also be of interest to control which objects are visible for the client.
You can do that with the `export` function. Note that export accepts regular
expressions which are indicated by a leading '^'.


```r
m <- module({
  
  export("fun")
  
  .privateFunction <- identity
  privateFunction <- identity
  fun <- identity
  
})

names(m)
```

```
## [1] "fun"
```

## Example: Modules as Parallel Process

One example where you may want to have more control of the enclosing environment 
of a function is when you parallelize your code. First consider the case when a 
*naive* implementation fails.


```r
library(parallel)
dependency <- identity
fun <- function(x) dependency(x) 

cl <- makeCluster(2)
clusterMap(cl, fun, 1:2)
```

```
## Error in checkForRemoteErrors(val): 2 nodes produced errors; first error: could not find function "dependency"
```

```r
stopCluster(cl)
```

To make the function `fun` self contained we can define it in a module. 


```r
m <- module({
  dependency <- identity
  fun <- function(x) dependency(x) 
})

cl <- makeCluster(2)
clusterMap(cl, m$fun, 1:2)
```

```
## [[1]]
## [1] 1
## 
## [[2]]
## [1] 2
```

```r
stopCluster(cl)
```


# Scripts as modules

You can make scripts into modules with `as.module` or implicitly when you refer
to a file in a call to `use`. Inside such a script you can use `import` and
`use` in the same way you typically use `library`. A major difference is, that
library will not only attach the stated package but also all packages in the 
depends field of that dependency. This is something you have to do manually
(explicitly) with `import`. Consider the following example where we create a
module in a temporary file with its dependencies.


```r
code <- "
import(methods)
import(aoos)
# This is an example:
list : generic(x) %g% standardGeneric('generic')
generic(x ~ ANY) %m% as.list(x)
"

fileName <- tempfile()
writeLines(code, fileName)
```

Then we can attach such a module into this session by the following:


```r
someModule <- use(fileName, attach = TRUE)
search()
```

```
##  [1] ".GlobalEnv"        "import:module"     "import:module"    
##  [4] "package:parallel"  "package:module"    "package:aoos"     
##  [7] "tools:rstudio"     "package:stats"     "package:graphics" 
## [10] "package:grDevices" "package:utils"     "package:datasets" 
## [13] "package:methods"   "Autoloads"         "package:base"
```

```r
generic(1:2)
```

```
## [[1]]
## [1] 1
## 
## [[2]]
## [1] 2
```


# Modules in Packages

...


# Modules with Object Orientation

S3 method dispatch will not work because of the special search mechanism of
`UseMethod`. By default the set functions of the methods package have side
effects in the top level environment. A module has only access to that
environment in a package. Most of the syntactic sugar (S4) in aoos works:


```r
m <- module({
  import("methods")
  import("aoos")
  gen(x) %g% cat("default method")
  gen(x ~ numeric) %m% cat("method for x ~ numerc")
})
m$gen("Hej")
```

```
## default method
```

```r
m$gen(1)
```

```
## method for x ~ numerc
```

S4 classes (or types in aoos) are not working correctly because it is terribly
hard to understand the side effects of the implementation. Until now I was not
able to isolate that. That means that up until now I have no clue how to define
a S4 class inside a module. `setOldClass` with an appropriate where statement
seems to work though:


```r
m <- module({
  import("methods")
  import("aoos")
  gen(x) %g% cat("default method")
  gen(x ~ numeric) %m% cat("method for x ~ numerc")
  NewType <- function(x) retList("NewType")
  setOldClass("NewType", where = environment())
  gen(x ~ NewType) %m% x
})

cl <- makeCluster(1)
clusterMap(cl, m$gen, list(m$NewType(NULL)))
```

```
## [[1]]
## $x
## NULL
## 
## attr(,".self")
## <environment: 0x1a3a3ef8>
## attr(,"class")
## [1] "NewType" "list"
```

```r
stopCluster(cl)
```
