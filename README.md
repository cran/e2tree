
<!-- README.md is generated from README.Rmd. Please edit that file -->

# Explainable Ensemble Trees (e2tree)

<!-- badges: start -->

[![R-CMD-check](https://github.com/massimoaria/e2tree/actions/workflows/R-CMD-check.yaml/badge.svg)](https://github.com/massimoaria/e2tree/actions/workflows/R-CMD-check.yaml)
[![CRAN
status](https://www.r-pkg.org/badges/version/e2tree)](https://CRAN.R-project.org/package=e2tree)
[![](http://cranlogs.r-pkg.org/badges/grand-total/e2tree)](https://cran.r-project.org/package=e2tree)

<!-- badges: end -->

<p align="center">

<img src="man/figures/e2tree_logo.png" width="400"  />
</p>

The **Explainable Ensemble Trees** (**e2tree**) key idea consists of the
definition of an algorithm to represent every ensemble approach based on
decision trees model using a single tree-like structure. The goal is to
explain the results from the esemble algorithm while preserving its
level of accuracy, which always outperforms those provided by a decision
tree. The proposed method is based on identifying the relationship
tree-like structure explaining the classification or regression paths
summarizing the whole ensemble process. There are two main advantages of
e2tree: - building an explainable tree that ensures the predictive
performance of an RF model - allowing the decision-maker to manage with
an intuitive structure (such as a tree-like structure).

In this example, we focus on Random Forest but, again, the algorithm can
be generalized to every ensemble approach based on decision trees.

## Setup

You can install the **developer version** of e2tree from
[GitHub](https://github.com) with:

``` r
install.packages("remotes")
remotes::install_github("massimoaria/e2tree")
```

You can install the **released version** of e2tree from
[CRAN](https://CRAN.R-project.org) with:

``` r
if (!require("e2tree", quietly=TRUE)) install.packages("e2tree")
```

``` r
require(e2tree)
require(randomForest)
require(ranger)
require(dplyr)
require(ggplot2)
if (!(require(rsample, quietly=TRUE))){install.packages("rsample"); require(rsample, quietly=TRUE)} 
options(dplyr.summarise.inform = FALSE)
```

## Warning

This package is still under development and, for the time being, the
following limitations apply:

- Only ensembles trained with the **randomForest** and **ranger**
  packages are currently supported. Support for additional packages and
  approaches will be added in the future.

- Currently **e2tree** works only for classification and regression
  problems. It will gradually be extended to handle other types of
  response variables, such as count data, multivariate responses, and
  more.

## Example 1: IRIS dataset

In this example, we want to show the main functions of the e2tree
package.

Starting from the IRIS dataset, we will train an ensemble tree using the
randomForest package and then subsequently use e2tree to obtain an
explainable tree synthesis of the ensemble classifier. We run a Random
Forest (RF) model, and then obtain the proximity matrix of the
observations as output. The idea behind the proximity matrix: if a pair
of observations is often at the a terminal node of several trees, this
means that both explain an underlying relationship. From this we are
able to calculate co-occurrences at nodes between pairs of observations
and obtain a matrix O of Co-Occurrences that will then be used to
construct the graphical E2Tree output. The final aim will be to explain
the relationship between predictors and response, reconstructing the
same structure as the proximity matrix output of the RF model.

``` r
# Set random seed to make results reproducible:
set.seed(0)

# Initialize the split
iris_split <- iris %>% initial_split(prop = 0.6)
iris_split
#> <Training/Testing/Total>
#> <90/60/150>
# Assign the data to the correct sets
training <- iris_split %>% training()
validation <- iris_split %>% testing()
response_training <- training[,5]
response_validation <- validation[,5]
```

Train an Random Forest model with 1000 weak learners

``` r
# Perform training with "ranger" or "randomForest" package:
## RF with "ranger" package
ensemble <- ranger(Species ~ ., data = training, num.trees = 1000, importance = 'impurity')

## RF with "randomForest" package
#ensemble = randomForest(Species ~ ., data = training, importance = TRUE, proximity = TRUE)
```

Here, we create the dissimilarity matrix between observations through
the createDisMatrix function

``` r
D = createDisMatrix(ensemble, data = training, label = "Species", parallel = list(active = FALSE, no_cores = NULL))
#> 
#> Attaching package: 'Rcpp'
#> The following object is masked from 'package:rsample':
#> 
#>     populate
```

setting e2tree parameters

``` r
setting=list(impTotal=0.1, maxDec=0.01, n=2, level=5)
```

Build an explainable tree for RF

``` r
tree <- e2tree(Species ~ ., data = training, D, ensemble, setting)
```

Plot the Explainable Ensemble Tree

``` r
expl_plot <- rpart2Tree(tree, ensemble)

# Plot using rpart.plot package:
plot_e2tree <- rpart.plot::rpart.plot(expl_plot,
                                      type=1,
                                      fallen.leaves = T,
                                      cex =0.55, 
                                      branch.lty = 6,
                                      nn = T, 
                                      roundint=F, 
                                      digits = 2,
                                      box.palette="lightgrey" 
                                      ) 
```

<img src="man/figures/README-unnamed-chunk-10-1.png" width="100%" />

Prediction with the new tree (example on training)

``` r
pred <- ePredTree(tree, training[,-5], target="virginica")
```

Comparison of predictions (training sample) of RF and e2tree

``` r
# "ranger" package
table(pred$fit, ensemble$predictions)
#>             
#>              setosa versicolor virginica
#>   setosa         33          0         0
#>   versicolor      0         23         3
#>   virginica       0          1        30

# "randomForest" package
#table(pred$fit, ensemble$predicted)
```

Comparison of predictions (training sample) of RF and correct response

``` r
# "ranger" package
table(ensemble$predictions, response_training)
#>             response_training
#>              setosa versicolor virginica
#>   setosa         33          0         0
#>   versicolor      0         22         2
#>   virginica       0          3        30

## "randomForest" package
#table(ensemble$predicted, response_training)
```

Comparison of predictions (training sample) of e2tree and correct
response

``` r
table(pred$fit,response_training)
#>             response_training
#>              setosa versicolor virginica
#>   setosa         33          0         0
#>   versicolor      0         25         1
#>   virginica       0          0        31
```

Variable Importance

``` r
V <- vimp(tree, training)
V
#> $vimp
#> # A tibble: 2 × 9
#>   Variable     MeanImpurityDecrease MeanAccuracyDecrease `ImpDec_ setosa`
#>   <chr>                       <dbl>                <dbl>            <dbl>
#> 1 Petal.Length                0.365             2.22e- 2            0.317
#> 2 Petal.Width                 0.214             1.41e-16           NA    
#> # ℹ 5 more variables: `ImpDec_ versicolor` <dbl>, `ImpDec_ virginica` <dbl>,
#> #   `AccDec_ setosa` <dbl>, `AccDec_ versicolor` <dbl>,
#> #   `AccDec_ virginica` <dbl>
#> 
#> $g_imp
```

<img src="man/figures/README-unnamed-chunk-15-1.png" width="100%" />

    #> 
    #> $g_acc

<img src="man/figures/README-unnamed-chunk-15-2.png" width="100%" />

Comparison with the validation sample

``` r
ensemble.pred <- predict(ensemble, validation[,-5])

pred_val<- ePredTree(tree, validation[,-5], target="virginica")
```

Comparison of predictions (sample validation) of RF and e2tree

``` r
## "ranger" package
table(pred_val$fit, ensemble.pred$predictions)
#>             
#>              setosa versicolor virginica
#>   setosa         17          0         0
#>   versicolor      0         26         0
#>   virginica       0          0        17

## "randomForest" package
#table(pred_val$fit, ensemble.pred$predicted)
```

Comparison of predictions (validation sample) of e2tree and correct
response

``` r
table(pred_val$fit, response_validation)
#>             response_validation
#>              setosa versicolor virginica
#>   setosa         17          0         0
#>   versicolor      0         24         2
#>   virginica       0          1        16
roc_res <- roc(response_validation, pred_val$score, target="virginica")
```

<img src="man/figures/README-unnamed-chunk-18-1.png" width="100%" />

``` r
roc_res$auc
#> [1] 0.9324973
```

To evaluate how well our tree captures the structure of the RF and
replicates its classification, we introduce a procedure to measure the
goodness of explainability. We start by visualizing the final partition
generated by the RF through a heatmap — a graphical representation of
the co-occurrence matrix, which reflects how often pairs of observations
are grouped together across the ensemble. Each cell shows a pairwise
similarity: the darker the cell, the closer to 1 the similarity —
meaning the two observations were frequently assigned to the same leaf.
Comparing these two matrices — both visually and statistically — allows
us to assess how well E2Tree reproduces the ensemble structure. To
formally test this alignment, we use the [Mantel
test](https://aacrjournals.org/cancerres/article/27/2_Part_1/209/476508/The-Detection-of-Disease-Clustering-and-a),
a statistical method that quantifies the correlation between the two
matrices. The Mantel test is a non-parametric method used to assess the
correlation between two distance or similarity matrices. It is
particularly useful when we are interested to study the relationships
between dissimilarity structures. The test uses permutation to generate
a null distribution, comparing the observed statistic against values
obtained under random reordering.

``` r
eComparison(training, tree, D, graph = TRUE)
```

<img src="man/figures/README-unnamed-chunk-19-1.png" width="100%" /><img src="man/figures/README-unnamed-chunk-19-2.png" width="100%" /><img src="man/figures/README-unnamed-chunk-19-3.png" width="100%" />

    #> $z.stat
    #> [1] 1046.257
    #> 
    #> $p
    #> [1] 0.001
    #> 
    #> $alternative
    #> [1] "two.sided"
