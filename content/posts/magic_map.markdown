---
title: Magic map() - a minimal functional programming workflow
author: Martin
date: '2021-03-19'
slug: []
categories: []
tags: ["rstats", "map", "functional_programming"]
description: ''
keywords: ["rstats", "map", "functional_programming"]
math: no
toc: no
---


"Form follows function" - _Louis Sullivan_

Functional programming is simply about writing code with functions. Instead of repeating the same line of code over and over or using double-nested for loops, we can abstract the essence of what we are doing into functions. A function can then be elegantly applied to many inputs. Here, we will do this with `purrr::map()`, which I'm using day in and day out and which is a great starting point to dive into the world of functional programming.

Let's go through some of the `map()`-magic with a minimal workflow to produce clean, robust and fast code. We will do a small genome-wide association study, an analysis looking at the association between genes and a trait by fitting a model over and over again for every SNP in the genome. 

Here are some my favorite packages.

```r
library(purrr) # provides the key function here: map
library(furrr) # does map in parallel
library(dplyr) # does all sorts of magic
library(glue)  # concatenates strings beautifully
library(broom) # takes a model and returns a tidy data.frame
```

Let's see whether drinking coffee has a genetic basis. We make up a trait 
(coffees per day) and 100 SNPs for 100 individuals. 

## Simulate data


```r
coffees   <- sample(1:6, 100, TRUE)
snps      <- replicate(100, sample(c(0,1,2), 100, TRUE))
snp_names <- paste0("snp", 1:100)

dat <- data.frame(cbind(coffees, snps)) %>% 
            setNames(c("coffees", snp_names))
            
head(dat[1:5, 1:7])
```

```
#>   coffees snp1 snp2 snp3 snp4 snp5 snp6
#> 1       1    0    1    0    0    2    1
#> 2       1    2    1    1    1    0    2
#> 3       1    0    0    2    2    0    0
#> 4       3    0    0    2    0    2    1
#> 5       3    1    2    2    2    0    0
```

## 1) Write a function

We could now do 100 linear models manually by writing 100 lines of code, or we could do a `for` loop. 

Instead, let's write a function to fit _one_ model, and then apply this function to each SNP. We generally want the thing that changes (i.e. snp_name) to be the first argument. The function below fits a linear model of coffee consumption with a snp as predictor, and extracts the model estimate and p-value for the SNP. It returns a one-row `data.frame`. I generally like my functions to return data.frames, because that makes it easy to put together many function outputs into a big data.frame at the end.


```r
fit_model <- function(snp_name, dat) {
      # write formula using SNP name
      model_formula <- glue("coffees ~ {snp_name}")
      # fit linear model
      fit <- lm(model_formula, data = dat) %>% 
                  broom::tidy() %>%        # tidy results
                  filter(term == snp_name) # extract snp
      return(fit)
      
}
```

## 2) Use `map()` to apply function to every SNP

Using a vector with `snp_names`, we can apply the function to every SNP. The structure of `map()` is always the same: map(list/vector, function, additional_arguments). `map()` always returns a list. We can convert the list to a data.frame with `dplyr::bind_rows()`.


```r
# run gwas
gwas <- map(snp_names, fit_model, dat) %>% 
              bind_rows()
# print first three SNPs
gwas[1:3, ]
```

```
#> # A tibble: 3 x 5
#>   term  estimate std.error statistic p.value
#>   <chr>    <dbl>     <dbl>     <dbl>   <dbl>
#> 1 snp1  -0.292       0.212   -1.38     0.172
#> 2 snp2  -0.00891     0.212   -0.0420   0.967
#> 3 snp3  -0.237       0.213   -1.11     0.270
```

## 3) What if something goes wrong? - `map()` safely

Loops often fail becomes something goes wrong in one or a few iterations.Let's introduce a non-existing SNP and try again to see how it fails


```r
snp_names2 <- c("not_a_snp", snp_names)

gwas <- map(snp_names2, fit_model, dat) %>% 
      bind_rows()
```

```
#> Error in eval(predvars, data, env): object 'not_a_snp' not found
```

We can make our gwas error-safe using `purrr::safely()`. This does some magic under the hood which isn't so important now. For every iteration, it will return a list with two elements, one for the result and one for the error (equals NULL if there is no error). This way, we always get the results or errors of all our iterations back.


```r
fit_model_safely <- purrr::safely(fit_model)

gwas <- map(snp_names2, fit_model_safely, dat)
gwas[1:2]
```

```
#> [[1]]
#> [[1]]$result
#> NULL
#> 
#> [[1]]$error
#> <simpleError in eval(predvars, data, env): object 'not_a_snp' not found>
#> 
#> 
#> [[2]]
#> [[2]]$result
#> # A tibble: 1 x 5
#>   term  estimate std.error statistic p.value
#>   <chr>    <dbl>     <dbl>     <dbl>   <dbl>
#> 1 snp1    -0.292     0.212     -1.38   0.172
#> 
#> [[2]]$error
#> NULL
```

`map()` lets you extract a list element simply by its name. Here, we iterate over the list of results and extract all SNPs that worked.

```r
gwas <- map(gwas, "result") %>% 
            bind_rows()
gwas[1:3, ]
```

```
#> # A tibble: 3 x 5
#>   term  estimate std.error statistic p.value
#>   <chr>    <dbl>     <dbl>     <dbl>   <dbl>
#> 1 snp1  -0.292       0.212   -1.38     0.172
#> 2 snp2  -0.00891     0.212   -0.0420   0.967
#> 3 snp3  -0.237       0.213   -1.11     0.270
```

## 4) What if it takes too long? - `map()` in parallel

Once the idea of `map()` is clear, we can easily parallelise it to run on multiple cores. There is some overhead in collecting computations from several cores so this doesn't make much sense when the running time is short. But for longer computations, using 4 cores instead of 1 should make it nearly 4 times as fast. 

We can use the `furrr` package here, which mimics `purrr` functions but can run in parallel. It is based on [future](https://cran.r-project.org/web/packages/future/vignettes/future-1-overview.html), which is why all functions start with `future_`, for example `future_map()`. Unlike other ways of parallelising, this approach works on Windows, Mac and Linux. All we have to do now is to first set up a `plan()` ...


```r
# check available cores
availableCores()
# parallelises across 4 cores
plan(multiprocess, workers = 4)
```

... and then replace `map()` with `future_map()`. 


```r
gwas <- future_map(snp_names, fit_model, dat) %>% 
            bind_rows()
gwas[1:3, ]
```

```
#> # A tibble: 3 x 5
#>   term  estimate std.error statistic p.value
#>   <chr>    <dbl>     <dbl>     <dbl>   <dbl>
#> 1 snp1  -0.292       0.212   -1.38     0.172
#> 2 snp2  -0.00891     0.212   -0.0420   0.967
#> 3 snp3  -0.237       0.213   -1.11     0.270
```

## The full, minimal workflow for a robust, parallel gwas


```r
plan(multiprocess, workers = 4)

fit_model <- function(snp_name, dat) {
      model_formula <- glue("coffees ~ {snp_name}")
      fit <- lm(model_formula, data = dat) %>% 
                  broom::tidy() %>% 
                  filter(term == snp_name)
      return(fit)
}

fit_model_safely <- purrr::safely(fit_model)
gwas <- future_map(snp_names, fit_model_safely, dat) %>% 
                  map("result") %>% 
                  bind_rows()

gwas[1:3, ]
```

```
#> # A tibble: 3 x 5
#>   term  estimate std.error statistic p.value
#>   <chr>    <dbl>     <dbl>     <dbl>   <dbl>
#> 1 snp1  -0.292       0.212   -1.38     0.172
#> 2 snp2  -0.00891     0.212   -0.0420   0.967
#> 3 snp3  -0.237       0.213   -1.11     0.270
```

## General thoughts about map

_When to use map?_

- Whenever more than two lines of code look similar, whenever a for loop needs two cups of coffee to be understood, map will be on your side

_Can every for-loop be replaced by a function and map?_

- I think yes, but it becomes less practical when an iteration depends on the previous iteration, though I rarely encounter that problem

_What about the base R functions apply, sapply, lapply?_

- lapply is very similar to map, though it lacks some very nice features which can be discovered over time. The other apply-functions can give surprising results, except for vapply, which is concise but slightly more complicated.

_What about map_df, map_lgl, map_dbl and all the other maps?_

- All these functions differ in what their output is. However, the list resulting from simple map() can easily be transformed into any of these. After getting to grips with map, all the other maps fall into place.

_What if I have more than one vector or list as input?_

- map2() takes two vectors as input and pmap() takes any number of vectors as input. I'm using pmap() to iterate over rows in a data.frame, but this is stuff for another blogpost I think.

## Where to go from here

* Jenny Bryan's [purrr tutorials](https://jennybc.github.io/purrr-tutorial/index.html)

* Hadley Wickham's ["The joy of functional programming"](https://www.youtube.com/watch?v=bzUmK0Y07ck&t=2589s])

* R4DS chapter on [iterations](https://r4ds.had.co.nz/iteration.html)


