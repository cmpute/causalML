CS7290 Causal Modeling in Machine Learning: Homework 2
======================================================

For this assignment, we will once again reason on a generative model
using `bnlearn` and `pyro`. Check out the [*bnlearn*
docs](http://www.bnlearn.com) and the [*pyro* docs](http://pyro.ai) if
you have questions about these packages.

Submission guidelines
---------------------

Use a Jupyter notebook and/or R Markdown file to combine code and text
answers. Compile your solution to a static PDF document(s). Submit both
the compiled PDF and source files. The TA's will recompile your
solutions, and a failing grade will be assigned if the document fails to
recompile due to bugs in the code. If you use [Google
Collab](https://colab.research.google.com/notebook), send the link as
well as downloaded PDF and source files.

Background information
----------------------

Recall the [survey data](survey.txt) discussed in the previous homework.

-   **Age (A):** It is recorded as *young* (**young**) for individuals
    below 30 years, *adult* (**adult**) for individuals between 30 and
    60 years old, and *old* (**old**) for people older than 60.
-   **Sex (S):** The biological sex of individual, recorded as *male*
    (**M**) or *female* (**F**).
-   **Education (E):** The highest level of education or training
    completed by the individual, recorded either *high school*
    (**high**) or *university degree* (**uni**).
-   **Occupation (O):** It is recorded as an *employee* (**emp**) or a
    *self employed* (**self**) worker.
-   **Residence (R):** The size of the city the individual lives in,
    recorded as *small* (**small**) or *big* (**big**).
-   **Travel (T):** The means of transport favoured by the individual,
    recorded as *car* (**car**), *train* (**train**) or *other*
    (**other**)

Travel is the *target* of the survey, the quantity of interest whose
behavior is under investigation.

We use the following directed acyclic graph (DAG) as our basis for
building a model of the process that generated this data.

![survey dag](survey.png)

Causal sufficiency assumptions
------------------------------

-   Build the DAG and name it `net`.

<!-- -->

    ## Warning: package 'bnlearn' was built under R version 3.5.2

    ## 
    ## Attaching package: 'bnlearn'

    ## The following object is masked from 'package:stats':
    ## 
    ##     sigma

Recall the assumptions of **faithfulness** and **minimality** that we
need to reason causally about a DAG. Let's test these assumptions.

First, run the following code block to create the `d_sep` function (this
is the same as the bnlearn's dsep function but removes some type
checking for purposes that will soon be apparent).

    d_sep <- bnlearn:::dseparation

The following code evaluates the d-separation statement "A is
d-separated from E by R and T". This statement is false.

    d_sep(bn = net, x = 'A', y = 'E', z = c('R', 'T'))

    ## [1] FALSE

We are going to do a brute-force evaluation of every possible
d-separation statement for this graph.

First, run the following code in R.

    vars <- nodes(net)
    pairs <- combn(x = vars, 2, list)
    arg_sets <- list()
    for(pair in pairs){
      others <- setdiff(vars, pair)
      conditioning_sets <- unlist(lapply(0:4, function(.x) combn(others, .x, list)), recursive = F)
      for(set in conditioning_sets){
        args <- list(x = pair[1], y = pair[2], z = set)
        arg_sets <- c(arg_sets, list(args)) 
      }
    }

The above code did a bit of combinatorics that calculates all the pairs
to compare, i.e. the 'x' and 'y' arguments in `d_sep`. For each pair,
all subsets of size 0 - 4 variables that are not in that pair are
calculated. Each pair / other variable subset combination is an element
in the `arg_sets` list.

For each pair of variables in the DAG, we want to evaluated if they are
d-separated by the other nodes in the DAG. The code abobe does a bit of
combinatorics to grab all pairs of variables from that DAG, and then for
each pair, calculates all subsets of size 0, 1, 2, 3, and 4 of other
variables that are not in that pair. For example, the arguments to the
above statement `d_sep(bn = net, x = 'A', y = 'E', z = c('R', 'T'))` are
the 10th element in that list:

    arg_sets[[10]]

    ## $x
    ## [1] "A"
    ## 
    ## $y
    ## [1] "E"
    ## 
    ## $z
    ## [1] "R" "T"

You can evaluate the satement as follows:

    arg_set <- arg_sets[[10]]
    d_sep(bn=net, x=arg_set$x, y=arg_set$y, z=arg_set$z)

    ## [1] FALSE

-   Create a new list. Iterate through the list of argument sets and
    evaluate if the d-separation statement is true. If a statement is
    true, add it to the list. Show code. Print an element from the list
    and write out the d-separation statement in English.

<!-- -->

    true_sets <- list()
    for(arg_set in arg_sets){
      if(d_sep(bn=net, x=arg_set$x, y=arg_set$y, z=arg_set$z)){
        true_sets <- c(true_sets, list(arg_set))
      }
    }

-   Given two d-separation statements A and B, if A implies B, then we
    can say B is a redundant statement. This list is going to have some
    redundant statements. Print out an example of two elements in the
    list, where one one element implies other element. Write both of
    them out as d-separation statements, and explain the redundancy in
    plain English.  
-   Based on this understanding of redundancy, how could this algorithm
    for finding true d-separation statements be made more efficient?

A DAG is minimal with respect to a distribution if
*U*⊥<sub>𝔾</sub>*W*|*V* ⇒ *U*⊥<sub>*P*<sub>𝕏</sub></sub>*W*|*V*, in
other words if every true d-separation statement in the DAG corresponds
to a conditional independence statement in the joint probability
distribution. We don't know the true underlying joint probability
distribution that generated this data, but we do have the data. That
means we can do statistical tests for conditional independence, and use
some quick and dirty statistical decision theory to decide whether a
conditional independence statement is true or false.

The `ci.test` function in `bnlearn` does statistical tests for
conditional independence. The null hypothesis in this test is that the
conditional independence is true. So our decision critera is going to
be:

> If p value is below a .05 significance threshold, conclude that the
> conditional independence statement is true. Otherwise conclude it is
> false.

    test_outcome <- ci.test('A', 'E', c('R', 'T'), .data)
    print(test_outcome)

    ## 
    ##  Mutual Information (disc.)
    ## 
    ## data:  A ~ E | R + T
    ## mi = 26.557, df = 12, p-value = 0.008945
    ## alternative hypothesis: true value is greater than 0

    print(test_outcome$p.value)

    ## [1] 0.00894518

    alpha <- .05
    print(test_outcome$p.value > alpha)

    ## [1] FALSE

-   Evaluate the causal minimality assumption by doing a conditional
    independence test for each true d-separation statement. Print the
    proportion of times you conclude the conditional independence
    statement is true. Print any statement where the p-value is not
    greater than .05.

<!-- -->

    for(arg_set in true_sets){
      test_result <- ci.test(arg_set$x, arg_set$y, arg_set$z, .data)
      if(test_result$p.value < .05){
        print(test_result)
      }
    }

    ## 
    ##  Mutual Information (disc.)
    ## 
    ## data:  A ~ O | E + S
    ## mi = 17.471, df = 8, p-value = 0.02556
    ## alternative hypothesis: true value is greater than 0
    ## 
    ## 
    ##  Mutual Information (disc.)
    ## 
    ## data:  A ~ R | E + O
    ## mi = 16.029, df = 8, p-value = 0.04197
    ## alternative hypothesis: true value is greater than 0
    ## 
    ## 
    ##  Mutual Information (disc.)
    ## 
    ## data:  O ~ S | A + E
    ## mi = 13.244, df = 6, p-value = 0.03932
    ## alternative hypothesis: true value is greater than 0
    ## 
    ## 
    ##  Mutual Information (disc.)
    ## 
    ## data:  O ~ S | E + T
    ## mi = 14.202, df = 6, p-value = 0.02746
    ## alternative hypothesis: true value is greater than 0
    ## 
    ## 
    ##  Mutual Information (disc.)
    ## 
    ## data:  S ~ T | E + O
    ## mi = 17.334, df = 8, p-value = 0.02681
    ## alternative hypothesis: true value is greater than 0

-   What can you say about these statements w.r.t the above question
    about redundancy?
-   Now evaluate the *faithfulness* assumption $U *{P*{} }W|V U \_{ }W|V
    $, or that every conditional independence statement that is true
    about the joint distribution corresponds to a d-separation in the
    graph. Iterate through the `arg_sets` list again, run the
    conditional independence test for each argument set, creating a new
    list of sets where you conclude the conditional independence
    statement is true.
-   What proportion of the true d-separation statements correspond to
    conclusions of conditional independence? What proportion of
    conclusions of conditional independence correspond to
    true-deseparation statements? How well do the faithfulness and
    minimality assumptions hold up with this DAG and dataset?