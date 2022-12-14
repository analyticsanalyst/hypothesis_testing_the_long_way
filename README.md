Hypothesis Testing the Long Way
================

``` r
options(scipen=999) 

library(tidyverse)
library(infer)
library(openintro)
```

### Notebook Objective

  - This notebook applies hypothesis testing methods using formulas and
    base R vs out of the box functions.
  - The spirit of this notebook is to “show your work” to build further
    intuition.

### Single Proportion Hypothesis Test

##### Use Case

  - Test a claim for a population proportion using a sample (i.e. do a
    majority of folks in the USA support new gun laws?)

##### Conditions

  - We check the following conditions in order to use the normal model
  - Observations are independent
  - Number of failures \>= 10 and number of successes \>= 10 (using null
    hypothesis proportion)

##### Example

  - Claim: this is a fair coin
  - Ho: p = 0.5
  - Ha: p \!= 0.5
  - Significance alpha level: 0.05

<!-- end list -->

``` r
### independence and success failure conditions pass
set.seed(121)
sample_size <- 50
p_null <- 0.5
alpha_significance_level <- 0.05

### generate a sample of 50 coin flips
coin <- c("heads", "tails")
coin_flips <- sample(coin, sample_size, replace = TRUE)

### observed sample proportion
p_hat_heads <- mean(coin_flips=="heads")

### derive standard error using p_null
se <- sqrt((p_null*(1-p_null))/sample_size)

### test stat for single prop
zstat <- (p_hat_heads - p_null) / se 

### two sided pvalue using the observed test stat
pvalue_coin_test <- pnorm(abs(zstat), lower.tail = FALSE) * 2
paste("Pvalue: ", round(pvalue_coin_test,18))
```

    ## [1] "Pvalue:  0.396143909152074"

``` r
paste("Pvalue below the alpha_significance_level:",
      pvalue_coin_test < alpha_significance_level)
```

    ## [1] "Pvalue below the alpha_significance_level: FALSE"

``` r
### method above ties with faster out of the box function approach
# prop.test(x = sum(coin_flips=="heads"),
#           n = sample_size,
#           p = 0.5,
#           correct = F)
```

##### Conclusion

  - We fail to reject the null hypothesis. In other words, we do not
    have strong enough evidence to suggest the heads coin flip
    proportion is different from 0.5.

### Grouped Observations Prop Hypothesis Test

##### Use Case

  - Test if two group proportions are different
  - Use sample data to infer about the population of interest

##### Conditions

  - We check the following conditions in order to use the normal model
  - Observations are independent between and within groups
  - Each group number of failures \>= 10 and number of successes \>= 10

##### Example

  - Is there a difference between proportion of females and males who
    received a college degree?
  - Ho: female degree proportion = male degree proportion
  - Ha: female degree proportion \!= male degree proportion
  - Significance alpha level: 0.05

<!-- end list -->

``` r
### data for demonstration only
### do not consider this data reliable
group_prop_df <- infer::gss %>%
      filter(year<=1985) %>%
      dplyr::select(sex, college)

table(group_prop_df)
```

    ##         college
    ## sex      no degree degree
    ##   male          61     16
    ##   female        38     14

``` r
round(prop.table(table(group_prop_df), margin = 1),2)
```

    ##         college
    ## sex      no degree degree
    ##   male        0.79   0.21
    ##   female      0.73   0.27

When difference in portions between groups has null diff of 0 then we
use a pooled proportion to check success failure condition and use the
pooled proportion for our standard error calc.

``` r
### in practice, we investigate data collection methods to determine independence
### in this example we're assuming independence condition checks out
### success and failure condition by group also checks out
pooled_phat <- group_prop_df %>% 
  summarise(degree_proportion = mean(college=="degree")) %>%
  pull(degree_proportion)

group_prop_df %>%
      group_by(sex) %>%
      summarise(success_count = n() * pooled_phat,
                failure_count = n() * (1-pooled_phat))
```

    ## # A tibble: 2 × 3
    ##   sex    success_count failure_count
    ##   <fct>          <dbl>         <dbl>
    ## 1 male            17.9          59.1
    ## 2 female          12.1          39.9

``` r
n1 <- group_prop_df %>%
            filter(sex=="male") %>%
            nrow()

n2 <- group_prop_df %>%
            filter(sex=="female") %>%
            nrow()

phat1 <- group_prop_df %>%
            filter(sex=="male") %>%
            summarise(degree_proportion = mean(college=="degree")) %>%
            pull(degree_proportion)

phat2 <- group_prop_df %>%
            filter(sex=="female") %>%
            summarise(degree_proportion = mean(college=="degree")) %>%
            pull(degree_proportion)

point_estimate <-  phat2 - phat1

se <- sqrt((pooled_phat*(1-pooled_phat))/n1 
           + (pooled_phat*(1-pooled_phat))/n2)

### test stat for single prop
zstat <- (point_estimate - 0) / se 

### two sided pvalue using the observed test stat
pvalue_group_props <- pnorm(abs(zstat), lower.tail = FALSE) * 2
paste("Pvalue: ", round(pvalue_group_props,18))
```

    ## [1] "Pvalue:  0.417811875267997"

``` r
paste("Pvalue below the alpha_significance_level:",
      pvalue_group_props < alpha_significance_level)
```

    ## [1] "Pvalue below the alpha_significance_level: FALSE"

``` r
### method above ties with faster out of the box function approach
# group_prop_matrix <- group_prop_df %>%
#             mutate(college = fct_rev(college)) %>%
#             table()
# prop.test(group_prop_matrix, correct = F)
```

##### Conclusion

  - Pvalue is not below the 0.05 significance level.
  - We do not have enough evidence to reject the null hypothesis in
    favor of the alternative hypothesis.
  - Note the wording above, we are not saying there is not a difference.
  - Rather, given the sample size and sample group proportions observed
    we do not have enough evidence to suggest the observed difference
    were unlikely under the null hypothesis distribution of no
    difference.

### Chi-square Goodness of Fit Test

##### Use Case

  - Categorical variable with three or more levels and we want to
    compare deviations of the level counts to expected counts

##### Conditions

  - Check the following conditions in order to use the chi distribution
    model
  - Observations that contribute to counts must be independent of other
    counts
  - Each count total must have at least 5 counts

##### Example

  - In a bag of Skittles, are the Skittle color counts different?
  - Ho: category level counts are equal
  - Ha: at least one category level count is not equal

<!-- end list -->

``` r
skittles_bag <- tibble(color=c("red", "yellow", "orange", "purple", "green"),
                       count = c(9, 9, 9, 11, 22),
                       expected_count = 60/5) %>%
      mutate(squared_difference_vs_expected = 
                   ((count-expected_count)^2) / expected_count)
skittles_bag
```

    ## # A tibble: 5 × 4
    ##   color  count expected_count squared_difference_vs_expected
    ##   <chr>  <dbl>          <dbl>                          <dbl>
    ## 1 red        9             12                         0.75  
    ## 2 yellow     9             12                         0.75  
    ## 3 orange     9             12                         0.75  
    ## 4 purple    11             12                         0.0833
    ## 5 green     22             12                         8.33

``` r
chi_stat <- skittles_bag %>%
      summarise(chi_test_stat = sum(squared_difference_vs_expected)) %>%
      pull()

### degree of freedom = k levels - 1
pvalue_chi_gof <- pchisq(chi_stat, df = length(skittles_bag$color)-1, lower.tail = FALSE)
paste("Pvalue: ", round(pvalue_chi_gof,18))
```

    ## [1] "Pvalue:  0.0305770166275991"

``` r
paste("Pvalue below the alpha_significance_level:",
      pvalue_chi_gof < alpha_significance_level)
```

    ## [1] "Pvalue below the alpha_significance_level: TRUE"

``` r
### results tie with out of the box function
# stats::chisq.test(x=skittles_bag$count,
#                    p=rep(1/5,5),
#                    correct=T)
```

##### Conclusion

  - We reject the null hypothesis in favor of the alternative
    hypothesis.
  - We have evidence to suggest there is a difference between the count
    of Skittles by color.

### Two Way Chi Square Test

##### Use Case

  - Determine if two variables are related in any way

##### Conditions

  - We check the following conditions in order to use the chi
    distribution model
  - Observations that contribute to counts must be independent of other
    counts
  - Each count total must have at least 5 counts

##### Example

  - Is there an association between treatment and patient outcomes?
  - Ho: no difference in the effectiveness of the treatments
  - Ha: at least one of the treatments performs differently than the
    others

<!-- end list -->

``` r
test_results <- 
tribble(
 ~Treatment, ~Success, ~Failure, 
  'Exercise', 111, 64,
  'Exercise+ProteinShake', 82, 80,
  'Exercise+ProteinShake+MagicPill', 76, 93
)
test_results
```

    ## # A tibble: 3 × 3
    ##   Treatment                       Success Failure
    ##   <chr>                             <dbl>   <dbl>
    ## 1 Exercise                            111      64
    ## 2 Exercise+ProteinShake                82      80
    ## 3 Exercise+ProteinShake+MagicPill      76      93

``` r
### note: likely a more efficient/cleaner way to do this
### expected count by cell = (row total * column total) / table total
expected_count <- test_results %>%
  mutate(table_total = sum(c_across(where(is.numeric)))) %>%
  rowwise(Treatment) %>%
  mutate(row_total = sum(c_across(Success:Failure))) %>%
  ungroup() %>%
  mutate_at(c("Success", "Failure"), 
            ~(row_total * sum(.))/table_total) %>%
  dplyr::select(-table_total, -row_total)

### derive chi square stat for two way table
chi_stat <- sum(((test_results[2:3] - expected_count[2:3])^2) / expected_count[2:3])

### two way table degrees of freedom: (# of rows - 1) * (# of columns - 1) 
dof <- (3 - 1) * (2 - 1)

pvalue_chi_test <- pchisq(chi_stat, df = dof, lower.tail = FALSE)
paste("Pvalue: ", round(pvalue_chi_test,18))
```

    ## [1] "Pvalue:  0.00204632560001819"

``` r
paste("Pvalue below the alpha_significance_level:",
      pvalue_chi_test < alpha_significance_level)
```

    ## [1] "Pvalue below the alpha_significance_level: TRUE"

``` r
### results tie with out of the box function
# test_results %>%
#       dplyr::select(Success, Failure) %>%
# stats::chisq.test()
```

##### Conclusion

  - Pvaule is below significance level therefore we reject the null
    hypothesis in favor of the alternative hypothesis.
  - We conclude that we have evidence to suggest at least one of the
    treatments performs differently than the others.

### Single Group Mean Hypothesis Test

##### Use Case

  - Test a claim for a population mean using a sample mean

##### Conditions

  - Sample observations are independent
  - Often a vague normality assumption used (rules of thumb below tend
    to be useful)
  - If our sample size less than 30 then we confirm there are no clear
    outliers and assume the data come from a nearly normal distribution
  - If our sample size is 30 or more then we confirm there are no
    extreme outliers

##### Example

  - Are Blizzard employee salaries equal to California average salaries?
  - For demonstration, let’s assume California average salaries are
    $60k. This is a fake example don’t share this we any news outlets.
  - Ho: CA Blizzard employee salaries = $60k
  - Ha: CA Blizzard employee salaries \!= $60k

<!-- end list -->

``` r
### https://www.openintro.org/data/index.php?data=blizzard_salary
blizzard_employee_salary <- 
      read_csv("https://www.openintro.org/data/csv/blizzard_salary.csv")

set.seed(111)
bes_sample <- blizzard_employee_salary %>%
      filter(status=="Full Time Employee",
             location == "Irvine",
             salary_type == "year",
             !is.na(current_salary),
             ### filter out dirty data records
             current_salary>1000) %>%
      sample_n(100)
```

##### Checking conditions:

1.  Independence: for demonstration purposes we will assume the sample
    is independent and representative of all California Blizzard
    employees.
2.  Normality: sample size is above 30 and no extreme outliers present
    (visual inspection via histogram and observation zscore inspection)

<!-- end list -->

``` r
### point estimate for the population average of interest
sample_mean <- mean(bes_sample$current_salary)

### standard error for the point estimate
sample_standard_error <- sd(bes_sample$current_salary) / 
      sqrt(length(bes_sample$current_salary))

### null hypothesis salary
null_salary <- 60000

### tstat we use for inference with means
tstat <- (sample_mean - null_salary) / sample_standard_error

### two sided pvalue using the observed test stat
pvalue_salary <- pt(abs(tstat), 
                    ### using t distribution vs normal distribution
                    ### although when n >= 30 then t dist is approximately normal
                    df=length(bes_sample$current_salary)-1,
                    lower.tail = FALSE) * 2
paste("Pvalue: ", round(pvalue_salary,18))
```

    ## [1] "Pvalue:  0"

``` r
paste("Pvalue below the alpha_significance_level:",
      pvalue_salary < alpha_significance_level)
```

    ## [1] "Pvalue below the alpha_significance_level: TRUE"

``` r
# ties with t test function
# t.test(x=bes_sample$current_salary,
#         mu=60000)$p.value
```

##### Conclusion

  - Reject the null in favor of the alternative hypothesis.
  - We have strong evidence that the average Blizzard employee salary is
    different from $60k.
  - Given the observed point estimate is above the null, we conclude
    that Blizzard employees have average salaries above $60k.
  - Remember this is a fake example.

### Two Groups Mean Hypothesis Test

##### Use Case

  - Test if there is a difference between population group means using
    sample data

##### Conditions

  - Sample observations are independent within and across groups
  - Outlier rules from one sample t test applied to each group

##### Example

  - Is there a difference between average home sale price between two
    neighborhoods?
  - Ho: North Ames Average Sale Price - College Creek Average Sale Price
    = 0
  - Ha: North Ames Average Sale Price - College Creek Average Sale Price
    \!= 0

<!-- end list -->

``` r
sample_group_size <- 60
set.seed(63)
example_price_data <- openintro::ames %>%
      filter(Neighborhood %in% c("NAmes", "CollgCr")) %>%
      group_by(Neighborhood) %>%
      sample_n(sample_group_size)
```

##### Checking conditions:

1.  Independence: for demonstration purposes we will assume the sample
    is independent within and across groups.
2.  Normality: sample size is above 30 in each group. Per group, no
    extreme outliers present (visual inspection via histogram and
    observation zscore inspection).

<!-- end list -->

``` r
na_sales_df <- example_price_data %>% filter(Neighborhood=="NAmes")
cc_sales_df <- example_price_data %>% filter(Neighborhood=="CollgCr")    

na_avg_price <- mean(na_sales_df$price)
na_sd_price <- sd(na_sales_df$price)

cc_avg_price <- mean(cc_sales_df$price)
cc_sd_price <- sd(cc_sales_df$price)

standard_error_diff_in_means <- sqrt(
      ((na_sd_price^2) / sample_group_size) + 
      ((cc_sd_price^2) / sample_group_size)
)

### 0 represents the null hypothesis diff
tstat <- ((na_avg_price - cc_avg_price) - 0) / standard_error_diff_in_means

### complex formula for degrees of freedom the long way
### in practice, we use out of the box functions for this
dof_numerator <- (((na_sd_price^2) / sample_group_size) + ((cc_sd_price^2) / sample_group_size))^2
dof_denominator <-  (((na_sd_price^2) / sample_group_size)^2) / (sample_group_size-1) + 
                    (((cc_sd_price^2) / sample_group_size)^2) / (sample_group_size-1)
dof <- dof_numerator / dof_denominator

### two sided pvalue using the observed test stat
pvalue_avg_price_diff <- pt(abs(tstat), 
                    df=dof,
                    lower.tail = FALSE) * 2
paste("Pvalue: ", round(pvalue_avg_price_diff,18))
```

    ## [1] "Pvalue:  0.00000000000004451"

``` r
paste("Pvalue below the alpha_significance_level:",
      pvalue_avg_price_diff < alpha_significance_level)
```

    ## [1] "Pvalue below the alpha_significance_level: TRUE"

``` r
# ties with t test function
# t.test(na_sales_df$price, cc_sales_df$price)
```

##### Conclusion

  - Pvalue is below significance level therefore we reject the null
    hypothesis in favor of the alternative hypothesis.
  - The data suggests there is a difference between average neighborhood
    home sale price.

### Three or More Group Means Hypothesis Test (ANOVA)

##### Use Case

  - Compare means across three or more groups

##### Conditions

  - Observations are independent within and across groups
  - Data within each group are nearly normal
  - Variability across groups are roughly equal

##### Example

  - Does average sepal length differ by Iris species?
  - Ho: group means are equal
  - Ha: at least one group means differ
  - For demonstration, we are assuming the conditions are met.
    Understanding how the data were collected and visual inspection tend
    to be the first pass checks of the ANOVA conditions. Statistical
    tests for normality and measures for variability might also be
    included.

<!-- end list -->

``` r
### F stat used for ANOVA test
### F stat = mean square between groups (MSG) / mean square error (MSE)
### MSG = SSG / (k - 1)
### MSE = SSE / (n - k)
### SSE = SST - SSG

k <- n_distinct(iris$Species)
n <- nrow(iris)
df1 <-  (k - 1)
df2 <- (n - k)

iris_setup <- iris %>%
      mutate(all_groups_mean_sep_length = mean(Sepal.Length),
             total_observations = n()) %>%
      group_by(Species) %>%
      mutate(group_mean_sep_length = mean(Sepal.Length)) %>%
      group_by(Species, 
               all_groups_mean_sep_length, 
               group_mean_sep_length,
               total_observations) %>%
      summarise(group_count = n()) %>%
      ungroup()

SSG <- iris_setup %>%
      summarise(ssg = sum(
            group_count * 
             ((group_mean_sep_length - all_groups_mean_sep_length)^2)
       )
      ) %>%
      pull()

SST <- iris %>%
      mutate(all_groups_mean_sep_length = mean(Sepal.Length)) %>%
      summarise(sst = sum((Sepal.Length - all_groups_mean_sep_length)^2)) %>%
      pull()

SSE <- SST - SSG
MSG <- SSG / df1
MSE <- SSE / df2

### test stat for ANOVA test
### above approach would be used in practice :)
### we can run this test with one line of code
fstat <- MSG / MSE

### fstat pvalue generated using f distribution
pvalue_fstat <- pf(q=fstat, df1 = df1, df2 = df2, lower.tail = F)

paste("Pvalue: ", round(pvalue_fstat,18))
```

    ## [1] "Pvalue:  0"

``` r
paste("Pvalue below the alpha_significance_level:",
      pvalue_fstat < alpha_significance_level)
```

    ## [1] "Pvalue below the alpha_significance_level: TRUE"

``` r
### ties with anova test
# anova(aov(Sepal.Length ~ Species, data=iris))$`Pr(>F)`[1]
# anova(aov(Sepal.Length ~ Species, data=iris))
```

##### Conclusion

  - We reject the null in favor of the alternative hypothesis.
  - We would conclude that 1 or more species groups have different
    average sepal lengths.

### Inspiration Source

  - [OpenIntro Statistics 4th
    Edition](https://www.openintro.org/book/os/)

### Future TODO

  - Add paired observation hypothesis tests
