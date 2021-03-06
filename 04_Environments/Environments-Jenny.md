# Jenny's reading of Environments
Jenny Bryan  
`r format(Sys.time(), '%d %B, %Y')`  



Week 04 2014-07-31 we read [Environments](http://adv-r.had.co.nz/Environments.html). Source is found [here on github](https://github.com/hadley/adv-r/blob/master/Environments.rmd).

## Taking the quiz

*I made myself enter these answers before reading the chapter, most especially before reading the answers. I annotated/corrected my original answers as I read on.*

#### List at least three ways that an environment is different to a list.

This will be rough sailing ... highly speculative:

  * You can't use the usual subsetting/indexing machinery with an environment, i.e. `joe_environment[[1]]`.
  * Functions like `length()`, `is.logical()`, etc. don't work on an environment.
  * That's all I've got.

*Here are the four important ways in which an environment is not like a list:*

  * *Every object in an environment has a unique name.*
  * *The objects in an environment are not ordered (i.e., it doesn’t make sense to ask what the first object in an environment is).*
  * *An environment has a parent.*
  * *Environments have reference semantics.*

#### What is the parent of the global environment? What is the only environment that doesn't have a parent?

  * I don't know.
  
*Pulling quotes from the chapter:*

  * *"The parent of the global environment is the last package that you attached with `library()` or `require()`."*
  * *"Only one environment doesn’t have a parent: the __empty__ environment."*
  
#### What is the enclosing environment of a function? Why is it important?

  * The environment in which the function was defined (or called?). It's important because that's where objects used in the function will be found if they are defined neither in the function body nor as formal arguments.

#### How do you determine the environment from which a function was called?

Apply the function `environment()` to the function?

#### How are `<-` and `<<-` different?

`<-` makes an assignment in the current environment, whereas `<<-` make an assignment in the parent environment (or is it the global environment?).

## Environment basics

Apparently we're going to use `pryr` alot, so I load it:


```r
library(pryr)
```

Notes....

### Exercises

#### List three ways in which an environment differs from a list.

  * "Every object in an environment has a unique name." *I am confused by this. There's a diagram that shows both `a` and `d` bound to the same object, a vector holding 1, 2, 3. So it seems like that object has 2 names. Discuss.*
  * "The objects in an environment are not ordered (i.e., it doesn’t make sense to ask what the first object in an environment is)." *So, as I guessed in the initial quiz, we can't select objects from an environment with, say, positive or negative integers or logical vectors.*
  * "An environment has a parent." *Lists don't have these family relationships.*

#### If you don't supply an explicit environment, where do `ls()` and `rm()` look? Where does `<-` make bindings?

  * `ls()` and `rm()` will look in the interactive workspace, aka the global environment.
  * That is also where `<-` makes bindings.

#### Using `parent.env()` and a loop (or a recursive function), verify that the ancestors of `globalenv()` include `baseenv()` and `emptyenv()`. Use the same basic idea to implement your own version of `search()`.

First, I use a loop to detail the ancestry of `globalenv()`. Warning: this is UGLY:

```r
library(plyr)
foo <- globalenv()
env_stuff <- vector("list", 15)
for(i in 1:15) {
  env_stuff[[i]] <- capture.output(str(foo))
  if(identical(foo, emptyenv())) break
  foo <- parent.env(foo)
  }
env_stuff <- ldply(env_stuff, "[", 1:2)
names(env_stuff) <- c('environment', 'name')
env_stuff <- mutate(env_stuff,
                    environment = gsub("environment: ", "", environment),
                    name = gsub(' - attr\\(\\*, \"name\"\\)= chr ', "", name))
env_stuff <- mutate(env_stuff, environment = gsub("[<>]", "", environment),
                    name = gsub('\"', '', name))
```

Here's the ancestry:


|environment        |name              |
|:------------------|:-----------------|
|R_GlobalEnv        |NA                |
|package:plyr       |package:plyr      |
|package:pryr       |package:pryr      |
|package:stats      |package:stats     |
|package:graphics   |package:graphics  |
|package:grDevices  |package:grDevices |
|package:utils      |package:utils     |
|package:datasets   |package:datasets  |
|package:methods    |package:methods   |
|0x100980268        |Autoloads         |
|base               |NA                |
|R_EmptyEnv         |NA                |

Yep, I see that the ancestors of `globalenv()` include `baseenv()` and `emptyenv()`.

Now I'll write a proper recursive function, i.e. write my own version of `search()`, and take advantage of `environmentName()`.


```r
j_search <- function(env = globalenv()) {
  if (identical(env, emptyenv())) {
    return(invisible(NULL))
    } else {
      return(c(environmentName(env), j_search(parent.env(env))))
      }
  }
j_search()
```

```
##  [1] "R_GlobalEnv"       "package:plyr"      "package:pryr"     
##  [4] "package:stats"     "package:graphics"  "package:grDevices"
##  [7] "package:utils"     "package:datasets"  "package:methods"  
## [10] "Autoloads"         "base"
```

```r
search()
```

```
##  [1] ".GlobalEnv"        "package:plyr"      "package:pryr"     
##  [4] "package:stats"     "package:graphics"  "package:grDevices"
##  [7] "package:utils"     "package:datasets"  "package:methods"  
## [10] "Autoloads"         "package:base"
```

Other than minor differences in name formatting, my function appears to be equivalent to `search()`.

## Recursing over environments

### Exercises

#### Modify `where()` to find all environments that contain a binding for `name`.


```r
j_where <- function(name, env = globalenv()) {
  if (identical(env, emptyenv())) {
    return(invisible(NULL))
    } else if (exists(name, envir = env, inherits = FALSE)) {
      return(c(environmentName(env), j_where(name, parent.env(env))))
      } else {
        j_where(name, parent.env(env))
        }    
  }
j_where("foo")
```

```
## [1] "R_GlobalEnv"
```

```r
j_where("plot")
```

```
## [1] "package:graphics"
```

```r
median <- "huh?"
j_where("median")
```

```
## [1] "R_GlobalEnv"   "package:stats"
```

```r
rm("median")
j_where("median")
```

```
## [1] "package:stats"
```

#### Write your own version of `get()` using a function written in the style of `where()`.


```r
j_get <- function(name, env = parent.frame()) {
  if (identical(env, emptyenv())) {
    # Base case
    stop("Can't find ", name, call. = FALSE)
    
  } else if (exists(name, envir = env, inherits = FALSE)) {
    # Success case
    return(eval(as.name(name), envir = env))
    
  } else {
    # Recursive case
    j_get(name, parent.env(env))
    
  }
}
get("+")
```

```
## function (e1, e2)  .Primitive("+")
```

```r
j_get("+")
```

```
## function (e1, e2)  .Primitive("+")
```

```r
get("i")
```

```
## [1] 12
```

```r
j_get("i")
```

```
## [1] 12
```


#### Write a function called `fget()` that finds only function objects. It should have two arguments, `name` and `env`, and should obey the regular scoping rules for functions: if there's an object with a matching name that's not a function, look in the parent. For an added challenge, also add an `inherits` argument which controls whether the function recurses up the parents or only looks in one environment.

#### Write your own version of `exists(inherits = FALSE)` (Hint: use `ls()`.) Write a recursive version that behaves like `exists(inherits = TRUE)`.

## Function environments

### The enclosing environment

### Binding environments

### Execution environments

### Exercises

#### List the four environments associated with a function. What does each one do? Why is the distinction between enclosing and binding environments particularly important?
    
#### Draw a diagram that shows the enclosing environments of this function:

#### Expand your previous diagram to show function bindings.

#### Expand it again to show the execution and calling environments.

#### Write an enhanced version of `str()` that provides more information about functions. Show where the function was found and what environment it was defined in.

## Binding names to values

### Exercises

#### What does this function do? How does it differ from `<<-` and why might you prefer it?

#### Create a version of `assign()` that will only bind new names, never re-bind old names. Some programming languages only do this, and are known as [single assignment laguages][single assignment].

#### Write an assignment function that can do active, delayed, and locked bindings. What might you call it? What arguments should it take? Can you guess which sort of assignment it should do based on the input?

## Explicit environments

### Avoiding copies

### Package state

### As a hashmap

## Quiz answers

#### There are four ways: every object in an environment must have a name; order doesn't matter; environments have parents; environments have reference semantics.
   
#### The parent of the global environment is the last package that you loaded. The only environment that doesn't have a parent is the empty environment.
    
#### The enclosing environment of a function is the environment where it was created. It determines where a function looks for variables.
    
#### Use `parent.frame()`.

#### `<-` always creates a binding in the current environment; `<<-` rebinds an existing name in a parent of the current environment.

[single assignment]:http://en.wikipedia.org/wiki/Assignment_(computer_science)#Single_assignment
