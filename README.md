<!-- README.md is generated from README.Rmd. Please edit that file -->
[![Build
Status](https://travis-ci.org/jonathan-g/datafsm.png?branch=master)](https://travis-ci.org/jonathan-g/datafsm)
[![CRAN\_Status\_Badge](http://www.r-pkg.org/badges/version-last-release/datafsm)](http://cran.r-project.org/package=datafsm)

This package implements our method for automatically generating models
of dynamic decision-making that both have strong predictive power and
are interpretable in human terms. We use an efficient model
representation and a genetic algorithm-based estimation process to
generate simple deterministic approximations that explain most of the
structure of complex stochastic processes. The genetic algorithm is
implemented with the **GA** package ([Scrucca
2013](http://www.jstatsoft.org/v53/i04/)). Our method, implemented in
C++ and R, scales well to large data sets. We have applied the package
to empirical data, and demonstrated the method's ability to recover
known data-generating processes by simulating data with agent-based
models and correctly deriving the underlying decision models for
multiple agent models and degrees of stochasticity.

A user of our package can estimate models by providing their data in a
common "panel data" format. The package is designed to estimate time
series classification models that use a small number of binary predictor
variables and move back and forth between the values of the outcome
variable over time. Larger sets of predictor variables can be reduced to
smaller sets with cross-validation. Although the predictor variables
must be binary, a quantitative variable can be converted into binary by
division of the observed values into high/low classes. Future releases
of the package may include additional estimation methods to complement
genetic algorithm optimization.

``` r
# Load and attach datafsm into your R session, making its functions available:
library(datafsm)
```

Please cite the package if you use it with the text generated by:

``` r
citation("datafsm")
#> 
#> To cite datafsm in publications use:
#> 
#>   John J. Nay, Jonathan M. Gilligan (2015) "Data-Driven Dynamic
#>   Decision Models," in L. Yilmaz et al. (eds.), Proc. 2015 Winter
#>   Simulation Conf.
#> 
#> A BibTeX entry for LaTeX users is
#> 
#>   @InProceedings{,
#>     title = {Data-Driven Dynamic Decision Models},
#>     author = {John J. Nay and Jonathan M. Gilligan},
#>     booktitle = {Proceedings of the 2015 Winter Simulation Conference},
#>     year = {2015},
#>     editor = {L. Yilmaz and W.K.V. Chan and I. Moon and T.M.K. Roeder and C. Macal and M. Rosetti},
#>   }
```

Fake Data Example
=================

To quickly show it works, we can create fake data. Here, we generate
1000 repetitions of a ten-round game in which each player starts by
making a random move, and in subsequent rounds, one player follows a
"tit-for-tat" strategy while the other one follows a "noisy tit-for-tat"
strategy that's equivalent to tit-for-tat, except that with a 10%
probability the player will make a random move.

``` r
seed1 <- 1900
seed2 <- 1900
seed3 <- 1900
# 468534172
set.seed(seed1)
cdata <- data.frame(outcome = NA,
                    period = rep(1:10, 2000),
                    my.decision1 = NA,
                    other.decision1 = NA)
#
# Prisoner's dilemma
#
pd_outcome <- function(player_1, player_2) {
  #
  # 1 = C
  # 2 = D
  #
  player_1  + 1
}

tit_for_tat <- function(last_round_self, last_round_opponent) {
    last_round_opponent
}

noise_level = 0.1

noisy_tit_for_tat <- function(last_round_self, last_round_opponent, noise_level) {
  if (runif(1,0,1) <= noise_level) {
    sample(0:1,1)
  } else {
    last_round_opponent
  }
}

for (i in seq_along(cdata$period)) {
  if (cdata$period[i] == 1) {
    my.decision <- sample(0:1,1, prob = c(1 - noise_level, noise_level))
    other.decision <- sample(0:1,1)
    cdata[i, "outcome"] <- pd_outcome(my.decision, other.decision)
  } else{
    my.last <- my.decision
    other.last <- other.decision
    my.decision <- tit_for_tat(my.last, other.last)
    other.decision <- noisy_tit_for_tat(other.last, my.last, noise_level)
    cdata[i,c("outcome", "my.decision1", "other.decision1")] <- 
      c(pd_outcome(my.decision, other.decision), my.last, other.last)
  }
}
```

The only required argument of the main function of the package,
`evolve_model`, is a `data.frame` object, which must have 3-5 columns.
The first two columns must be named "period" and "outcome" (period is
the time period that the outcome action was taken). The remaining one to
three columns are predictors, and may have arbitrary names. Each row of
the `data.frame` is an observational unit, an action taken at a
particular time and any relevant variables for that time. All of the
(3-5 columns) should be named. The period and outcome columns should be
integer vectors---e.g. `c(1,2,1)`---and the columns with the predictor
variable data should be logical vectors---e.g. `c(TRUE, FALSE, FALSE)`---or 
vectors that can be coerced to logical with `as.logical()`.

Here are the first eleven rows of this fake data:

|  outcome|  period|  my.decision1|  other.decision1|
|--------:|-------:|-------------:|----------------:|
|        1|       1|            NA|               NA|
|        1|       2|             0|                0|
|        1|       3|             0|                0|
|        1|       4|             0|                0|
|        1|       5|             0|                0|
|        1|       6|             0|                0|
|        1|       7|             0|                0|
|        1|       8|             0|                0|
|        1|       9|             0|                0|
|        1|      10|             0|                0|
|        1|       1|            NA|               NA|

We can estimate a model with that data. `evolve_model` uses a genetic
algorithm to estimate a finite-state machine (FSM) model, primarily for
understanding and predicting decision-making. This is the main function
of the `datafsm` package. It relies on the `GA` package for genetic
algorithm optimization. We chose to use a GA because GAs perform well in
rugged search spaces to solve integer optimization problems, are a
natural complement to our binary representation of FSMs, and are easily
parallelized.

`evolve_model` takes data on predictors and data on the outcome and
automatically creates a fitness function that takes training data, an
action vector that `evolve_model` generates, and a state matrix
`evolve_model` generates as input and returns numeric vector whose
length is the number of rows in the training data. `evolve_model` then
computes a fitness score for that potential solution FSM by comparing it
to the provided `outcome` in the real training data. This is repeated
for every FSM in the population and then the probability of selection
for the next generation is proportional to the fitness scores. If the
argument `cv` is set to `TRUE` the function will call itself recursively
while varying the number of states inside a cross-validation loop in
order to estimate the optimal number of states, then it will set the
number of states to that optimal number and estimate the model on the
full training set.

If the argument `parallel` is set to `TRUE`, then these evaluations are
distributed across the available processors of the computer using the
`doParallel` functions; otherwise, the evaluations of fitness are
conducted sequentially. Because this fitness function that
`evolve_model` creates must loop through all the training data every
time it is evaluated and we need to evaluate many possible solution
FSMs, the fitness function is implemented in C++ to improve its
performance.

``` r
set.seed(seed2)
res <- evolve_model(cdata, seed = seed3)
```

`evolve_model` evolves the models on training data and then, if a test
set is provided, uses the best solution to make predictions on test
data. Finally, the function returns the GA object and the decoded
version of the best string in the population. Formally, the function
return an `S4` object with slots for:

-   `call` Language from the call of the function.
-   `actions` Numeric vector with the number of actions.
-   `states` Numeric vector with the number of states.
-   `GA` S4 object created by `ga()` from the `GA` package.
-   `state_mat` Numeric matrix with rows as states and columns as
    predictors.
-   `action_vec` Numeric vector indicating what action to take for each
    state.
-   `predictive` Numeric vector of length one with test data accuracy if
    test data was supplied; otherwise, a character vector with a message
    that the user should provide test data for better estimate of
    generalizable performance.
-   `varImp` Numeric vector with length equal to the number of columns
    in the state matrix, containing relative importance scores for each
    predictor.
-   `timing` Numeric vector length one containing the total elapsed time
    it took `evolve_model` to execute.
-   `diagnostics` Character vector length one, designed to be printed
    with `base::cat()`.

Use the summary and plot methods on the output of the `evolve_model()`
function:

``` r
summary(res)
#> Length  Class   Mode 
#>      1 ga_fsm     S4
plot(res, action_label = ifelse(action_vec(res)==1, "C", "D"), 
     transition_label = c('cc','dc','cd','dd'))
```

![Result of `plot()` method call on `ga_fsm`
object.](man/figures/README/plot.fsm-1.png)

The diagram shows that `evolve_model` recovered a tit-for-tat model in
which the player in question ("me") mimics the last action of the
opponent.

Use the `estimation_details` method on the output of the
`evolve_model()` function:

``` r
suppressMessages(library(GA))
plot(estimation_details(res))
```

![Result of `plot()` method call on ga object, which is obtained by
calling `estimation_details()` on `ga_fsm`
object.](man/figures/README/plot.evolution-1.png)

Acknowledgements
================

This work is supported by U.S. National Science Foundation grants
EAR-1416964 and EAR-1204685.

Session Info
============

``` r
sessionInfo()
#> R version 3.4.4 (2018-03-15)
#> Platform: x86_64-pc-linux-gnu (64-bit)
#> Running under: Ubuntu 16.04.5 LTS
#> 
#> Matrix products: default
#> BLAS: /usr/lib/libblas/libblas.so.3.6.0
#> LAPACK: /usr/lib/lapack/liblapack.so.3.6.0
#> 
#> locale:
#>  [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
#>  [3] LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
#>  [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
#>  [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                 
#>  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
#> [11] LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       
#> 
#> attached base packages:
#> [1] stats     graphics  grDevices utils     datasets  methods   base     
#> 
#> other attached packages:
#> [1] GA_3.1.1         iterators_1.0.10 foreach_1.4.4    datafsm_0.2.2   
#> 
#> loaded via a namespace (and not attached):
#>  [1] magic_1.5-8         ddalpha_1.3.4       tidyr_0.8.1        
#>  [4] sfsmisc_1.1-2       splines_3.4.4       prodlim_2018.04.18 
#>  [7] assertthat_0.2.0    highr_0.7           stats4_3.4.4       
#> [10] DRR_0.0.3           yaml_2.1.19         robustbase_0.93-1.1
#> [13] ipred_0.9-6         diagram_1.6.4       pillar_1.3.0       
#> [16] backports_1.1.2     lattice_0.20-35     glue_1.3.0         
#> [19] digest_0.6.15       colorspace_1.3-2    recipes_0.1.3      
#> [22] htmltools_0.3.6     Matrix_1.2-14       plyr_1.8.4         
#> [25] timeDate_3043.102   pkgconfig_2.0.1     CVST_0.2-2         
#> [28] broom_0.5.0         caret_6.0-80        purrr_0.2.5        
#> [31] scales_0.5.0        gower_0.1.2         lava_1.6.2         
#> [34] tibble_1.4.2        ggplot2_3.0.0.9000  withr_2.1.2        
#> [37] nnet_7.3-12         lazyeval_0.2.1      cli_1.0.0          
#> [40] survival_2.42-6     magrittr_1.5        crayon_1.3.4       
#> [43] evaluate_0.11       doParallel_1.0.11   nlme_3.1-137       
#> [46] MASS_7.3-50         dimRed_0.1.0        class_7.3-14       
#> [49] tools_3.4.4         stringr_1.3.1       kernlab_0.9-26     
#> [52] munsell_0.5.0       bindrcpp_0.2.2      pls_2.6-0          
#> [55] compiler_3.4.4      RcppRoll_0.3.0      rlang_0.2.1        
#> [58] grid_3.4.4          rmarkdown_1.10      geometry_0.3-6     
#> [61] gtable_0.2.0        ModelMetrics_1.1.0  codetools_0.2-15   
#> [64] abind_1.4-5         reshape2_1.4.3      R6_2.2.2           
#> [67] lubridate_1.7.4     knitr_1.20          dplyr_0.7.6        
#> [70] bindr_0.1.1         rprojroot_1.3-2     shape_1.4.4        
#> [73] stringi_1.2.4       parallel_3.4.4      Rcpp_0.12.18       
#> [76] rpart_4.1-13        DEoptimR_1.0-8      tidyselect_0.2.4
```
