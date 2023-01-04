Proportional, grouped histograms and density plots
================
Donald T. McKnight (<donald.mcknight@my.jcu.edu.au>)

-   [1 TL:DR](#1-tldr)
-   [2 Topics covered](#2-topics-covered)
-   [3 Introduction/the problem](#3-introductionthe-problem)
-   [4 Example data](#4-example-data)
-   [5 Histograms](#5-histograms)
    -   [5.1 Solutions](#51-solutions)
        -   [5.1.1 Grouped histogram where each group sums to
            1](#511-grouped-histogram-where-each-group-sums-to-1)
            -   [5.1.1.1 Why this works](#5111-why-this-works)
        -   [5.1.2 Grouped histogram where all groups collectively sum
            to
            1](#512-grouped-histogram-where-all-groups-collectively-sum-to-1)
            -   [5.1.2.1 Why this works](#5121-why-this-works)
    -   [5.2 Things that don’t work](#52-things-that-dont-work)
        -   [5.2.1 Default geom_histogram](#521-default-geom_histogram)
        -   [5.2.2 scales::percent_format()](#522-scalespercent_format)
        -   [5.2.3 ..density..](#523-density)
-   [6 Density plots](#6-density-plots)
    -   [6.1 Solutions](#61-solutions)
        -   [6.1.1 Grouped density plot where each group sums to
            1](#611-grouped-density-plot-where-each-group-sums-to-1)
            -   [6.1.1.1 Why this works](#6111-why-this-works)
        -   [6.1.2 Grouped density plot where all groups combined sum to
            1](#612-grouped-density-plot-where-all-groups-combined-sum-to-1)
            -   [6.1.2.1 Why this works](#6121-why-this-works)
        -   [6.1.3 Adding a mean line to a grouped density
            plot](#613-adding-a-mean-line-to-a-grouped-density-plot)
    -   [6.2 Things that don’t work](#62-things-that-dont-work)
        -   [6.2.1 Default geom_density](#621-default-geom_density)
        -   [6.2.2 ..scaled..](#622-scaled)
-   [7 Combing histograms and density
    plots](#7-combing-histograms-and-density-plots)
    -   [7.1 Combine plots where each group sums to
        1](#71-combine-plots-where-each-group-sums-to-1)
    -   [7.2 Combine plots where all data sums to
        1](#72-combine-plots-where-all-data-sums-to-1)

# 1 TL:DR

-   For grouped histograms where each group sums to 1 (e.g., two species
    on the same plot):
    -   in geom_histogram use: **aes(y=stat(width \* density))**
-   For grouped histograms where all groups combined sum to 1 (e.g.,
    plotting males and females within a population):
    -   in geom_histogram use: **aes(y=..count../sum(..count..))**
-   For grouped density plots where each group sums to 1 (e.g., two
    species on the same plot):
    -   in geom_density use: **aes(y=binwidth \* ..density..)**
        -   binwidth is the binwidth you would use if making a histogram
            (not the same as bw)
            -   binwidth is not a recognized variable name in
                geom_density. Simply enter it as a number or define it
                beforehand (e.g., aes(y=2\*..density..) for binwidth=2)
-   For grouped density plots where all groups combined sum to 1 (e.g.,
    plotting males and females within a population):
    -   in geom_density use: **aes(y=binwidth \* proportions \*
        ..density..)**
        -   binwidth is the binwidth you would use if making a histogram
            (not the same as bw)
            -   binwidth is not a recognized variable name in
                geom_density. Simply enter it as a number or define it
                beforehand (e.g., aes(y=2 \* proportions \* ..density..)
                for binwidth=2)
        -   proportions is the proportion of points belonging to each
            group
            -   this is a vector where each entry is the proportion of
                points for that group (for each group, the length needs
                to equal n in geom_density (default is 512))
                -   e.g., for two groups consisting of 15 and 5 points
                    (respectively), proportions =
                    c(rep(.75,512),rep(.25,512))
            -   I have provided a function (get.props) to make this easy

# 2 Topics covered

-   Grouped histograms displayed as proportions
-   Grouped density plots (smoothed histograms) displayed as proportions
-   Combining histograms and density plots at the same scale
    (proportions)
-   Adding mean lines (across groups) to grouped density plots

# 3 Introduction/the problem

-   When making histograms and density plots, we often want the Y axis
    scaled to represent proportions (or percentages), because this
    standardizes histograms and curves across groups and puts things on
    a scale that is intuitive to interpret.
-   ggplot (for all its benefits) does not have an intuitive (IMO)
    method for doing this when you have more than one group, and the
    solutions that either work for single groups or seem like they
    should work can give very misleading and inaccurate results.
-   Solving this for histograms (via geom_histogram) is fairly easy
    (though finding the solution online was very difficult), but I was
    utterly unable to find a solution for density plots (via
    geom_density), despite finding many others who were struggling with
    the same issue.
-   After much trial and error, I eventually found a solution that seems
    to work
-   This document outlines the issue, presents the code and examples for
    my solution, and explains why many commonly proposed solutions don’t
    actually work (but often deceptively look like they work).

**NOTE** I have not done much with these solutions while also using
facet_wrap(). I do not know if they will work in that situation
(particularly for grouped density plots where all groups collectively
sum to 1)

**Disclaimer** I have done my best to test all of this and make sure
that it works, but obviously I’m not providing any grantees. Also, there
may be more elegant solutions out there, this is just the best that I
could come up with.

![](E:/todays%20files/23-10-22/it-does-run.jpg)

# 4 Example data

-   Load packages
-   Make a data frame (“df”) with two groups (“a” and “b”)
-   Groups have similarly-shaped distributions, but the median is
    different and “a” is a bit flatter
-   “a” also has twice the sample size of “b”

``` r
df <- cbind.data.frame(values = c(75,97,92,95,88,87,89,76,81,83,35,57,52,55,48,37,39,36,50,43,50,57,52,55,48,37,35,36,41,43),id = c(rep("b",10),rep("a",20)))

library(ggplot2) #version 3.3.5
library(dplyr) #version 1.0.8
library(plyr) #version 1.8.6
library(scales) #version 1.1.1 this is only needed to illustrate one of the faulty "solutions"
```

# 5 Histograms

## 5.1 Solutions

### 5.1.1 Grouped histogram where each group sums to 1

-   This solution is for cases where you are trying to plot two or more
    independent groups on the same plot and each group should sum to 1.
-   For example, you might use this if “a” and “b” in our example are
    two species, and the values are the sizes of individuals.
-   **Solution:** add aes(y=stat(width\*density)) in geom_histogram
-   This will calculate the proportions per group
-   First I am going to show it using only the strictly necessary code

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_histogram(aes(y=stat(width*density)))
```

    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

![](Tutorial_git_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

-   This worked, but the defaults are pretty awful
-   I will add the following arguments and use them throughout
    -   binwidth = the width of the bins into which data are grouped
    -   boundary = sets the start position of the first bin
    -   alpha = sets a transparency
    -   color = sets the color of the border around each bar
    -   scale_x\_continuous = sets the limits on the x axis
    -   labs = labels the y axis

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_histogram(binwidth=10,boundary=0,alpha=0.5,color="black",aes(y=stat(width*density)))+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

-   Just to double check, let’s increase the binwidth to 20

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_histogram(binwidth=20,boundary=0,alpha=0.5,color="black",aes(y=stat(width*density)))+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

-   Still works
-   For clarity, the sum of each group is now 1 (thus the sum of the
    entire graph is 2)

#### 5.1.1.1 Why this works

-   Briefly, stat(density) and ..density.. are calculating kernel
    density estimates *not* probabilities. Thus, the integral of the
    density will always sum to 1, but individual points along the
    density cloud can be greater than 1.
-   To get the probability estimate (which can be treated as a
    proportion) you need to integrate the values over some length along
    the x axis.
-   This is where width argument comes in, and why multiplying the width
    by the density gives you a probability (which you can view as a
    proportion)

### 5.1.2 Grouped histogram where all groups collectively sum to 1

-   This solution is for situations where you are plotting related
    groups, each of which is a proportion of a whole (i.e., the sum of
    all groups = 1)
    -   For example, if our groups “a” and “b” are males and females,
        and the values are sizes, you might want to look at the
        distribution of sizes in the entire population while
        distinguishing males and females
-   **Solution:** add aes(y=..count../sum(..count..)) in geom_histogram

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_histogram(binwidth=10,boundary=0,alpha=0.5,color="black",aes(y=..count../sum(..count..)))+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

-   For clarity, the sum of the entire graph is now 1. So the number of
    data points per group affects the results
-   **Note** this will also work if you just have a single group
-   **Warning** I saw this solution come up a lot online when people
    were asking about grouped histograms where *each* group sums to 1
    rather than *all* groups collectively summing to 1.
    -   This does not work in the former situation (see previous section
        for how to make that graph)

#### 5.1.2.1 Why this works

-   The ..x.. statements (e.g., ..count..) can be confusing
-   The are just internal names ggplot is using for its graph data
    -   You can access them by making your graph an object then using
        ggplot_build() over the name of your object then using the
        dollar sign to
    -   This will give you a data frame of values ggplot is using, and
        it is honestly a good thing to play with to really understand
        what ggplot is up to
-   In this case, for each bin, ..count.. has the number of values in
    that bin
    -   So sum(..count..) gives you the total number of values in the
        data set
    -   Thus, dividing the number of values in each bin (..count..) by
        the total number of values in the data set (sum(..count..))
        converts each bin to a proportion rather than a count extract
        the data object

``` r
p <- ggplot(df,aes(x=values,fill=id))+
  geom_histogram(binwidth=10,boundary=0,alpha=0.5,color="black",aes(y=..count../sum(..count..)))+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")

ggplot_build(p)$data
```

    ## [[1]]
    ##       fill          y count   x xmin xmax density    ncount  ndensity
    ## 1  #F8766D 0.00000000     0   5    0   10   0.000 0.0000000 0.0000000
    ## 2  #F8766D 0.00000000     0  15   10   20   0.000 0.0000000 0.0000000
    ## 3  #F8766D 0.00000000     0  25   20   30   0.000 0.0000000 0.0000000
    ## 4  #F8766D 0.23333333     7  35   30   40   0.035 1.0000000 1.0000000
    ## 5  #F8766D 0.23333333     7  45   40   50   0.035 1.0000000 1.0000000
    ## 6  #F8766D 0.20000000     6  55   50   60   0.030 0.8571429 0.8571429
    ## 7  #F8766D 0.00000000     0  65   60   70   0.000 0.0000000 0.0000000
    ## 8  #F8766D 0.06666667     0  75   70   80   0.000 0.0000000 0.0000000
    ## 9  #F8766D 0.16666667     0  85   80   90   0.000 0.0000000 0.0000000
    ## 10 #F8766D 0.10000000     0  95   90  100   0.000 0.0000000 0.0000000
    ## 11 #F8766D 0.00000000     0 105  100  110   0.000 0.0000000 0.0000000
    ## 12 #F8766D 0.00000000     0 115  110  120   0.000 0.0000000 0.0000000
    ## 13 #00BFC4 0.00000000     0   5    0   10   0.000 0.0000000 0.0000000
    ## 14 #00BFC4 0.00000000     0  15   10   20   0.000 0.0000000 0.0000000
    ## 15 #00BFC4 0.00000000     0  25   20   30   0.000 0.0000000 0.0000000
    ## 16 #00BFC4 0.00000000     0  35   30   40   0.000 0.0000000 0.0000000
    ## 17 #00BFC4 0.00000000     0  45   40   50   0.000 0.0000000 0.0000000
    ## 18 #00BFC4 0.00000000     0  55   50   60   0.000 0.0000000 0.0000000
    ## 19 #00BFC4 0.00000000     0  65   60   70   0.000 0.0000000 0.0000000
    ## 20 #00BFC4 0.06666667     2  75   70   80   0.020 0.4000000 0.4000000
    ## 21 #00BFC4 0.16666667     5  85   80   90   0.050 1.0000000 1.0000000
    ## 22 #00BFC4 0.10000000     3  95   90  100   0.030 0.6000000 0.6000000
    ## 23 #00BFC4 0.00000000     0 105  100  110   0.000 0.0000000 0.0000000
    ## 24 #00BFC4 0.00000000     0 115  110  120   0.000 0.0000000 0.0000000
    ##    flipped_aes PANEL group       ymin       ymax colour size linetype alpha
    ## 1        FALSE     1     1 0.00000000 0.00000000  black  0.5        1   0.5
    ## 2        FALSE     1     1 0.00000000 0.00000000  black  0.5        1   0.5
    ## 3        FALSE     1     1 0.00000000 0.00000000  black  0.5        1   0.5
    ## 4        FALSE     1     1 0.00000000 0.23333333  black  0.5        1   0.5
    ## 5        FALSE     1     1 0.00000000 0.23333333  black  0.5        1   0.5
    ## 6        FALSE     1     1 0.00000000 0.20000000  black  0.5        1   0.5
    ## 7        FALSE     1     1 0.00000000 0.00000000  black  0.5        1   0.5
    ## 8        FALSE     1     1 0.06666667 0.06666667  black  0.5        1   0.5
    ## 9        FALSE     1     1 0.16666667 0.16666667  black  0.5        1   0.5
    ## 10       FALSE     1     1 0.10000000 0.10000000  black  0.5        1   0.5
    ## 11       FALSE     1     1 0.00000000 0.00000000  black  0.5        1   0.5
    ## 12       FALSE     1     1 0.00000000 0.00000000  black  0.5        1   0.5
    ## 13       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5
    ## 14       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5
    ## 15       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5
    ## 16       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5
    ## 17       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5
    ## 18       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5
    ## 19       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5
    ## 20       FALSE     1     2 0.00000000 0.06666667  black  0.5        1   0.5
    ## 21       FALSE     1     2 0.00000000 0.16666667  black  0.5        1   0.5
    ## 22       FALSE     1     2 0.00000000 0.10000000  black  0.5        1   0.5
    ## 23       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5
    ## 24       FALSE     1     2 0.00000000 0.00000000  black  0.5        1   0.5

## 5.2 Things that don’t work

### 5.2.1 Default geom_histogram

-   Here is the default histogram
-   It’s not “wrong,” but it is potentially misleading (depending on the
    question) because the sample size for “a” is larger than the sample
    size for “b”
-   Also, we want this as proportions

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_histogram(binwidth=10,boundary=0,alpha=0.5,color="black")+
  scale_x_continuous(limits=c(0,120))
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

### 5.2.2 scales::percent_format()

-   I see this one as a solution very frequently, and it simply does not
    work (maybe it did in an old version?)
-   The solution claims that you simply add one of the following to your
    code:
    -   scale_y\_continuous(labels = scales::percent_format())
    -   scale_y\_continuous(labels = percent, name = “percent”)
-   The danger is that it often *looks* like it worked, but if you
    double check, the numbers are often bogus.
-   Using our data, the issue is obvious, but it’s not always this clear

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_histogram(binwidth=10,boundary=0,alpha=0.5,color="black")+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")+
  scale_y_continuous(labels = scales::percent_format())
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

-   Obviously, we should not be having bins labeled \>100%

### 5.2.3 ..density..

-   I have included this one because I have seen it show up as a
    solution, and while it *sometimes* works, it **does not work in all
    situations**, but you can mistakenly convince yourself it does work
    with some dummy data sets
-   The issue is that it is making a kernel density estimate histogram,
    not a simple count histogram. As a result, whether or not the
    density sums to 1 is entirely dependent on the binwidth and aspects
    of the data (whereas the integral of the area under the curve will
    always sum to 1)
-   To demonstrate this, let’s divide our data by 10 and run it with a
    binwidth of 1 and aes(y=..density..)

``` r
ggplot(df,aes(x=values/10,fill=id))+
  geom_histogram(binwidth=1,boundary=0,alpha=0.5,aes(y=..density..),color="black")+
  scale_x_continuous(limits=c(0,12))+
  labs(y="Proportion")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

-   That *looks* like it worked. All the “a” bars sum to 1, and all the
    “b” bars sum to 1.
-   But watch what happens if we set binwidth to 2 instead of 1

``` r
ggplot(df,aes(x=values/10,fill=id))+
  geom_histogram(binwidth=2,boundary=0,alpha=0.5,aes(y=..density..),color="black")+
  scale_x_continuous(limits=c(0,12))+
  labs(y="Proportion")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

-   For both “a” and “b” the sum of the bars is now 0.5, which is not
    what we want
-   Likewise, if we run our original data (rather than the data/10) with
    a binwidth of 10, the x axis is off by an order of magnitude

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_histogram(binwidth=10,boundary=0,alpha=0.5,aes(y=..density..),color="black")+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

-   So despite being a common answer online, this **does not work** in
    most cases and should not be used
-   This one is particularly dangerous, IMO, because it often *looks*
    like it worked

# 6 Density plots

-   The goal here is fundamentally to make a smoothed histogram. The
    problem is that geom_density is producing a density estimate *not* a
    probability
-   As with the histograms, we need to integrate using some width (the
    spread along the x)
-   Irritatingly, the trick we used for histograms does not work, but
    something similar does

## 6.1 Solutions

### 6.1.1 Grouped density plot where each group sums to 1

-   This solution is for cases where you are trying to plot two or more
    independent groups on the same plot and each group should sum to 1.
-   For example, you might use this if “a” and “b” in our example are
    two species, and the values are the sizes of individuals.
-   **Solution:** add aes(y=binwidth\*..density..) in geom_density
    -   binwidth is a number and should be the binwidth you would use if
        making a histogram (not the same as bw)

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_density(bw=5,alpha=0.5,color="black",aes(y=10*..density..))+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-12-1.png)<!-- -->

-   You can prove to yourself that this works by combining it with a
    histogram (see section 7)

#### 6.1.1.1 Why this works

-   If you think back to the histograms, to go from a density to a
    probability (which we can treat as a proportion), we need to
    multiply the density by some width value
-   Therefore, in geom_density() you can set
    aes(y=binwidth\*..density..) where binwidth is the width of the bins
    you would want if this was a histogram.
    -   **NOTE** binwidth is not an inherent argument in ggplot; it is
        just a variable name I am using as a placeholder (I am calling
        it binwidth because it is often best to set this value to the
        binwidth value you would want in a histogram)
    -   binwidth is not inherently the same as the bw argument in
        geom_density()
    -   bw is used internally for geom_density to set the smoothing
        bandwith
    -   leaving bw on default is usually fine, but I personally find
        that setting it to 1/2 binwidth makes the most sensible graphs

### 6.1.2 Grouped density plot where all groups combined sum to 1

-   This solution is for cases where your groups are not independent and
    are actually a part of some whole, so you want the groups
    distinguished, but you want the sum of all values to equal 1 (e.g.,
    examining size distributions of males and females in a population)
-   We need to use the binwidth\*..density.. solution described earlier
    with one additional step to account for sample size differences
    -   In short, for each group we need to take binwidth\*density, and
        multiply it by the proportion of all data points belonging to
        that group
    -   I have written the get.props() function below to do this, and it
        can be integrated into your ggplot code
    -   Note that it requires the package plyr
-   **NOTE** be careful if you reorder your factors. That may require
    some adjustments to this code depending on where you did the
    reordering (basically, make sure the order of proportions following
    get.props() matches the ordering of factors in your graph)

``` r
#x = vector of grouping data in the original data entered into ggplot
#n = n from the geom_density() function, default is 512

#returns the proportion of all points belonging to each group
get.props <- function(x,n=512){
   rep(plyr::count(x)[,2]/sum(plyr::count(x)[,2]),each=n)}


ggplot(df,aes(x=values,fill=id))+
  geom_density(bw=5,alpha=0.5,color="black",aes(y=10*..density..*get.props(df$id)))+ #same as before except we have added the get.props function and are using it on the id column of the original data frame
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

-   You can prove to yourself that this worked by combing it with
    histograms (see section 7)

#### 6.1.2.1 Why this works

-   This is just an extension of the solution we used for grouped
    density plots where each group sums to 1
-   In that first solution, we calculated a density per group, each of
    which summed to 1
-   If we want to convert that to a plot where all groups collectively
    sum to 1, we just have to scale each group by the proportion of
    points that belong to it
-   That simply requires multiplying the density distribution per group
    by the proportion of points belonging to that group.
    -   The get.props() function provides those proportions

### 6.1.3 Adding a mean line to a grouped density plot

-   In some cases, you may want to add a mean line that shows the
    average value across all of your groups
-   This **is not** as simple as just running geom_density without
    specifying groups because that will fail to take the differences in
    sample size into account
-   The best solution I have been able to come up with is to:
    1.  Make a simplified graph
    2.  Extract the data from the graph using ggplot_build()
    3.  Average those data
    4.  Add them back into the graph using geom_line()

    -   **NOTE** you must multiply the y values by the binwidth (see
        earlier code) in geom_line for this to plot correctly
-   I have provided the function get.mean.density to simplify this
-   First let’s make our starting graph
-   I’m going to use a different data set to demonstrate this

``` r
set.seed(1234)
df2 <- cbind.data.frame(values=c(rnorm(300,100,2),rnorm(100,92.5,1),rnorm(400,95,2),rnorm(200,90,1.5)),id=c(rep("a",300),rep("b",100),rep("c",400),rep("d",200)))

p <- ggplot()+
  geom_density(df2,bw=1,alpha=0.5,color="black",mapping=aes(x=values,fill=id,y=2*..density..))+
  scale_x_continuous(limits=c(80,110))+
  labs(y="Proportion")

p
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-14-1.png)<!-- -->

-   Now we can use the function below to calculate the mean line (dplyr
    is required)

``` r
# p = initial density plot (a gpplot object)
# n.groups = the number of groups in p
#n = n from geom_density, default is 512

get.mean.density <- function(p,n.groups,n=512){
  p.data <- as.data.frame(ggplot_build(p)$data) #extract data from graph
  p.data$bin <- rep(1:512,n.groups) #add a column of bin information based on n
  p.data.mean <- p.data %>% group_by(bin) %>% dplyr::summarize(x=x[1],y=mean(density)) #get averages
  p.data.mean}
```

-   Run the function

``` r
mean.density <- get.mean.density(p=p,n.groups=4)
```

-   Add to plot as a line

``` r
ggplot()+
  geom_density(df2,bw=1,alpha=0.5,color="black",mapping=aes(x=values,fill=id,y=2*..density..))+ #binwidth = 2
  scale_x_continuous(limits=c(80,110))+
  labs(y="Proportion")+
  geom_line(mean.density,mapping=aes(x=x,y=2*y),size=2) #note that y is multiplied by 2 because that is the binwidth used above
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-17-1.png)<!-- -->

-   Just to prove that this worked the way we wanted it to, let’s modify
    our data (df2) so that the sample sizes are the same per group
    -   This will let us make a proportional histogram for the entire
        data set (without specifying the groups) that should match our
        mean line
    -   This is the case because the mean line took sample size into
        account by first calculating the proportional density line per
        group, then averaging those lines

``` r
# modify data so that sample sizes are the same per group
set.seed(1234)
df3 <- cbind.data.frame(values=c(rnorm(400,100,2),rnorm(400,92.5,1),rnorm(400,95,2),rnorm(400,90,1.5)),id=c(rep("a",400),rep("b",400),rep("c",400),rep("d",400)))

#make proportional histogram with those data (no groups specified) and include our mean line from before
ggplot()+
  geom_histogram(df3,binwidth=1,alpha=0.5,mapping=aes(x=values,y=stat(width*density)))+
  scale_x_continuous(limits=c(80,110))+
  labs(y="Proportion")+
  geom_line(mean.density,mapping=aes(x=x,y=y),size=2) #note that y does not need to be multiplied by anything because binwidth is 1
```

    ## Warning: Removed 2 rows containing missing values (geom_bar).

![](Tutorial_git_files/figure-gfm/unnamed-chunk-18-1.png)<!-- -->

-   Note that it is not a 100% perfect match, but that is to be expected
    from smoothed density lines compared to block histograms
-   So far what I have shown seems pretty ugly and pointless, so let me
    try to give a nice illustration of the type of thing you can do with
    this
-   If, for example, you have 8 largely overlapping groups, and you want
    to show the central tendency and the variation, you can easily do so
    by plotting each group as the same color with an alpha, and plotting
    the mean line as a different color (in this case, yellow)

``` r
#make data
set.seed(1234)
df4 <- cbind.data.frame(values=c(rnorm(300,100,1.8),rnorm(100,95,3),rnorm(400,95,2.5),rnorm(200,97,3.5),rnorm(300,100,2.3),rnorm(100,98,2.1),rnorm(400,98,1.75),rnorm(200,99,1.85)),id=c(rep("a",300),rep("b",100),rep("c",400),rep("d",200),rep("e",300),rep("f",100),rep("g",400),rep("h",200)))

#make initial plot
p2 <- ggplot()+
  geom_density(df4,bw=1,alpha=0.25,color="black",mapping=aes(x=values,fill=id,y=2*..density..))+
  scale_x_continuous(limits=c(80,110))

#get mean density values
mean.density2 <- get.mean.density(p=p2,n.groups=8)

#make final plot
ggplot()+
  geom_density(df4,bw=1,alpha=0.12,color=NA,mapping=aes(x=values,fill=id,y=2*..density..))+
  scale_x_continuous(limits=c(80,110))+
  scale_y_continuous(expand=c(0,0),limits=c(0,.4))+
  labs(y="Proportion")+
  geom_line(mean.density2,mapping=aes(x=x,y=2*y),size=2,color="gold")+
  scale_fill_manual(values=rep("blue",8))+
  theme_bw()+
  theme(panel.grid.major = element_blank(),panel.grid.minor=element_blank(),legend.position="none")
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

## 6.2 Things that don’t work

### 6.2.1 Default geom_density

-   The default geom_density can be a bit misleading sometimes because
    it can give the appearance that things worked.
-   As an example, let’s run the default, but divide our data by 10
-   First lets try just running geom_density()

``` r
ggplot(df,aes(x=values/10,fill=id))+
  geom_density(bw=.5,alpha=0.5,color="black")+
  scale_x_continuous(limits=c(0,12))
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

-   That deceptively looks like it worked and we just need to rename the
    y axis, but it is an illusion. Or, more accurately, it worked on
    this particular data set, but will not work on a great many others.
-   To illustrate, let’s run it on the original data without diving by
    10

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_density(bw=5,alpha=0.5,color="black")+
  scale_x_continuous(limits=c(0,120))
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->

-   Notice that the shape of the curves has not changed (as expected
    from simply moving the decimal point).
-   However, the y-axis no longer matches proportions and it is now
    difficult for mere mortals to interpret.
-   This is because in both cases, the y is the kernel density; it just
    happens that in some cases that aligns with proportions, but that
    **is not** always the case. To ensure it matches proportions, we
    have to integrate over some length

### 6.2.2 ..scaled..

-   I have frequently seen this proposed as a solution, but it **does
    not** do what we are after.
-   It scales each group from 0-1, but that actually distorts the
    differences between groups and in no way helps with the
    interpretation

``` r
ggplot(df,aes(x=values,fill=id))+
  geom_density(bw=5,alpha=0.5,color="black",aes(y=..scaled..))+
  scale_x_continuous(limits=c(0,120))
```

![](Tutorial_git_files/figure-gfm/unnamed-chunk-22-1.png)<!-- -->

-   As you can see, this makes the shape of “a” and “b” look more
    similar than they actually are, because *both* groups now scale from
    0-1, even though the peak for “b” should actually be higher than the
    peak for “a”
-   Further, the y axis still is not particularly helpful (if anything,
    it is even more confusing now, IMO, because the scaled values look
    like proportions at a quick glance, even though they clearly aren’t)

# 7 Combing histograms and density plots

-   You may, in some cases, want that smoothed density line plotted over
    your histograms
-   This is easy to do by simply combining the solutions for each plot
    type
-   It is also a useful sanity check to make sure the density plots are
    doing what you expect

## 7.1 Combine plots where each group sums to 1

-   Add the smoothed density lines over each histogram
    -   Note that I have set boundary back to default to ensure that the
        plots line up

``` r
ggplot(df,aes(x=values,fill=id))+
  #histogram portion
  geom_histogram(binwidth=10,alpha=0.5,color="black",mapping=aes(y=stat(width*density)))+
  geom_density(size=2,alpha=0,mapping=aes(y=10*..density..,color=id))+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")
```

    ## Warning: Removed 4 rows containing missing values (geom_bar).

![](Tutorial_git_files/figure-gfm/unnamed-chunk-23-1.png)<!-- -->

## 7.2 Combine plots where all data sums to 1

-   Add the smoothed density lines over each histogram
    -   Note that I have set boundary back to default to ensure that the
        plots line up

``` r
ggplot(df,aes(x=values,fill=id))+
  #histogram portion
  geom_histogram(binwidth=10,alpha=0.5,color="black",mapping=aes(y=..count../sum(..count..)))+
  geom_density(size=2,alpha=0,mapping=aes(y=10*..density..*get.props(df$id),color=id))+
  scale_x_continuous(limits=c(0,120))+
  labs(y="Proportion")
```

    ## Warning: Removed 4 rows containing missing values (geom_bar).

![](Tutorial_git_files/figure-gfm/unnamed-chunk-24-1.png)<!-- -->
