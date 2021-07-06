[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **SDA_2019_St_Gallen_RDD_political_support** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml

Name of Quantlet: 'SDA_2019_St_Gallen_RDD_political_support'

Published in: 'SDA_2019_St_Gallen'

Description: 'Introduces the reader to regression discontinuity design, ”RDD”, by means of replicating ”Government Transfers and Political Support” by Manacorda, Miguel, Vigorito (2009). RDD is an econometric technique to establish causal effects based on the idea that institutional cut off criteria can be seen as arbitrarily, and hence near randomly, imposed on subjects in an area close to the predefined cut off criteria.'

Keywords: 'Regression discontinuity design, causal inference, econometrics, government transfers, political support'

Author: 'Filip Waldemar Mellgren, filip.mellgren@gmail.com.'

Submitted:   '24 November 2019, Filip Mellgren'

Datafile: 'reg_panes.dta, mccrary.dta, chic_aug_07.csv'

Input: 'Survey information on voting intention, eligibility of a government program and more'

Output:  'SDA_2019_St_Gallen_RDD_political_support.html'

```

### RMD Code
```rmd

---
title: "SDA 2019 St Gallen: Regression Discontinuity Design and role of government transfers on political support"
author: "Filip Mellgren"
date: '2019-11-23'
output:
  html_document: default
  pdf_document: default
bibliography: bibliography.bib
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```


```{r libraries, include=FALSE}
library(rio)
library(tidyverse)
library(locfit)
library(rddensity)
library(rdd)
library(stargazer)
library(AER)
library(ivpack)
# Set a parameter depending on preffered output
stargazer_output <- "html" 
# stargazer_output <- "text" 
#stargazer_output <- "latex" 

```
This document presents the topic of regression discontinity design, an econometric tehnnique used to establish causal estimates when the data varies discontinuously around a point due to some institutional feature. The original intention was to do this on Airbnb data collected from the web, however, I proceed with replicating a paper because there were no apparent discontinutiy present in the Airbnb data. Results from the Airbnb data are presented towards the end.

# Replicating Manacorda, Miguel, Vigorito (2009)

The paper by @manacorda2011government can be accessed from here:
http://emiguel.econ.berkeley.edu/assets/miguel_research/17/_WorkingPaper__Government__Transfers__and__Political_Support.pdf

Abstract of the paper:

> "We estimate the impact of a large anti-poverty cash transfer program, the Uruguayan PANES, on political
support for the government that implemented it. Using the discontinuity in program assignment based
on a pre-treatment eligibility score, we find that beneficiary households are 11 to 14 percentage points
more likely to favor the current government relative to the previous government. Political support
effects persist after the program ends. A calibration exercise indicates that these persistent impacts
are consistent with a model of rational but poorly informed voters learning about politicians’ redistributive
preferences."

It strives to establish causal effects of government benefits on government support. The following is noted regarding the current academic debate about the issue:

* There is an empirically established relationship that voters respond to macro economic trends.
* At the same time, there are econometric concerns about endogeneity and small, aggregated samples.
* The true, causal, magnitude is unknown.

In their paper, the authors wishes to establish a causal link between political support of the incumbent party and government transfers. The setting is such that families receive a government program called "PANES" if their predicted income is above a certain threshold. The government bases its support on a prediction because reported income may suffer from being unstable following a major crisis in the country in the foremath of the event, hard to verify claims as the target often worked informally, and multiple variables help mitigate strategic misreporting.

In summary the Panes program looked had the following defining features:

* It was a temporary cash transfer and food voucher following an economic downturn in the year 2002
* Recipients were households with low predicted income "based on a large number of pre-treatment covariates".
    - following crisis, many had an unstable income, so present was bad predictor of permanent income
    - hard to verify income claims as the target populaiton of worked informaly
    - multiple variables helps against strategic misreporting
* Households around the predicted income threshold for receiving Panes were "surveyed and asked a series of questions including their support for the current government, and a second similar follow-up survey took place the following year".


```{r load_data}
df_mccrary <- import("mccrary.dta")
df_reg_panes <- import("reg_panes.dta")
```

```{r data_manipulation}
# Create variables indicating whether above or below threshold
# Later cleans the h_89 variable
df_reg_panes %>% mutate(above = if_else(ind_reest > 0, ind_reest, NaN),
                        below = if_else(ind_reest < 0, ind_reest, NaN),
                        gov2007 = case_when(
                          h_89 == 9 ~NaN,
                          h_89 == NA ~NaN,
                          TRUE ~ (h_89-1)/2)
                        ) -> df
```

## Descriptive Statistics

To begin the analysis, I establish some observational facts by estimating a regression using OLS. Of course, these results will contain endogeneity bias but will nonetheless be useful as a benchmark. Potential reasons for endogeneity might include:

* Recipients already leaned towards one political side.
* Recipients respond they approve of the president in order to not loose benefits
    - However, Uruguay isn't a corrupt country, hence unlikely.h

```{r exploration,results='asis'}
# Simple regression, approval status on approved for PANES
df2 <- df %>% rename("Gov. Support" = gov2007, "Received PANES"=aprobado, "Predicted Income" = ind_reest)
ols_fit <- lm(`Gov. Support` ~ `Received PANES` + `Predicted Income`, data = df2)
stargazer(ols_fit, type = stargazer_output, flip = TRUE)
```

Overall, there is a negative relationship between predicted income and government support. Likely reflecting party difference in support among various income groups.

## Exogenous variation
To escape from the endogeneity present in the previous analysis, we can think of predicted income as a variable with some exogenous variation around the threshold of being assigned the government support. The basic idea is that households on either side of the threshold will be similar in all characteristics but their treatment status. 

Falling slightly to the right means some indicators of income predicts the household is above the threshold whereas falling on the left side of the threshold means the predcitors state that the household is poor enough to receive the benefits. However, except for extremely modest variation in the income indicators, predicted income may be deemed as something varying almost entirely randomly. 

If there happens to be a large discontinuity in government support around the specified threshold, the effect can plausibly be attributed to treatment status as nothing else may be expected to vary around the somewhat arbtirary threshold of predicted income.

There are, however, a few issues that must be dealt with. First, if certain households (or government officials) anticipate that they may be close to the threshold, they might try to affect the measure predicted income so as to move themselves to the left and thus increase their chance of receiving the government benefit resulting in a potnetially biased sample where the treated population is skewed towards households capable of affecting the outcomes of the government transfer, A second issue relate to potential confounding variables that also vary around the threshold. For example, if it turns out that predicted income was also used for some other government program, then it'd be unclear from what program potential effects adhere. Another issue might be if the metric varies around the threshold based on some observable characteristics, then effects could depend on this characteristic rather than treatment status. 

These issues, however, can be checked. The first issue implies a skewed density distirbution around the threshold whereas the second needs to make the case that this variable exists and then show a discontinuity precisly around the threshold. In practice, it is hard to come up with such variables and especially in this case because predicted income is a construct of several variables and unlikely to be used elsewhere.

A final concern partains to whether there is a discontinuity or just a sharp (continuous) decline. This will be an issue if the so called "bandwidth" is wide (see the three images below) and influenced by observations far from the cutoff that makes a continuous function appear discontinuous. 

There is a connection between a regression discontinuity estimates and instrumental variables (in fact, a fuzzy RDD is an IV estimation, @hahn2001identification was first with this interpretation). I therefore proceed by presenting the analysis as if it was an IV together with the rich set of intuitive graphs present in an RDD estimation.

## First stage
The first stage in an IV study considers the strength of the exogenous instrument $z$ (referred to as the "instrument"") in predicting the endogenous variable of interest $x$. Ideally, the instrument is a strong predictor of the endogenous variable to avoid issues related to weak variable bias. See @bound1995problems for an explanation of the problem with weak instruments. Other than establishing a tight link between the two variables, the first stage is simply a mechanic check that needs to be done. It's importance is measured by an F statistic which is ideally larger than 10.

```{r, message=FALSE, warning=FALSE}
# First stage
df %>% gather(above, below, key = "Treatment", value = "Predicted_income") %>% 
  ggplot(aes(x = Predicted_income, y = aprobado, 
             color = Treatment, fill = Treatment)) +
  geom_smooth(method = "loess", span = 0.5) +
  geom_point(size = 0.3) +
  theme_minimal() +
  labs(title = " Discontiuity around threshold", 
       x = "Predicted income", y = "Ever received PANES") +
    theme(
        panel.background = element_rect(fill = "transparent",colour = NA),
        plot.background = element_rect(fill = "transparent",colour = NA)
        )
ggsave("images/first_stage.png", bg = "transparent")
```
As can be seen in the figure above, there is a clear relationship between the two. Namely that when predicted income changes status, so does PANES status. This is intuitively clear because the government is only handing out the PANES to eligible households (those poor according to predicted income). There are a few exceptions but overall, the relationship is very strong which in the words of RDD can be referred to as a "Sharp RDD".

## Reduced form, support for government

The reduced form can also be interpreted as the intention to treat effect, people on one side of the cutoff were intended to receive PANES and the following graphs show how people intended to receive panes vary their support for the government. The ITT effect is the difference between the graphs around the cutoff.

An important hyperparameter to consider is what bandwidth is selected for the analysis. As can be seen, the local linear regression estimates (the curvy graphs) produce noisier estimates than does the linear relationship. The reason is that they are locally estimated using less data. In principle it is good to establish the causal effects with an as small interval as possible as this leads to less bias. Unfortunately, it also increases variance which is why there has to be a trade off between the two. I show three variations to give a feel for the sensitivity. 


```{r, message=FALSE, warning=FALSE}
# Reduced form: support for govt - first wave
# Lower span is less biased, but a noisier estimate.

df %>% 
  gather(above, below, key = "Treatment", value = "Predicted_income") %>% 
  ggplot(aes(x = Predicted_income, y = gov2007, 
             color = Treatment, fill = Treatment)) +
  geom_smooth(method = lm) +
  theme_minimal() +
  labs(title = "Linear estimate", 
       x = "Predicted income", y = "Government support 2007") +
    theme(
        panel.background = element_rect(fill = "transparent",colour = NA),
        plot.background = element_rect(fill = "transparent",colour = NA)
        )
ggsave("images/reduced_line.png", bg = "transparent")


df %>% 
  gather(above, below, key = "Treatment", value = "Predicted_income") %>% 
  ggplot(aes(x = Predicted_income, y = gov2007, 
             color = Treatment, fill = Treatment)) +
  geom_smooth(method = "loess", span = 0.33) +
  theme_minimal() +
  labs(title = "Span = 0.33", 
       x = "Predicted income", y = "Government support 2007") +
    theme(
        panel.background = element_rect(fill = "transparent",colour = NA),
        plot.background = element_rect(fill = "transparent",colour = NA)
        )
ggsave("images/reduced_0.33.png", bg = "transparent")

df %>% 
  gather(above, below, key = "Treatment", value = "Predicted_income") %>% 
  ggplot(aes(x = Predicted_income, y = gov2007, 
             color = Treatment, fill = Treatment)) +
  geom_smooth(method = "loess", span = 0.67) +
  theme_minimal() +
  labs(title = "Span = 0.67", 
       x = "Predicted income", y = "Government support 2007") +
    theme(
        panel.background = element_rect(fill = "transparent",colour = NA),
        plot.background = element_rect(fill = "transparent",colour = NA)
        )
ggsave("images/reduced_0.67.png", bg = "transparent")
  
```

## Iv regression
In this part, I formally estimate a model already shown in the graphs based on chapter 6 in @angrist2008mostly.

The model is as follows: 

* First stage: $D_i = \gamma_0 + \gamma_1 x_i + \pi T_i + \gamma_2 x_i T_i + \xi_{1i}$
* Reduced form, ITT: $Y_i = \mu + \kappa_1 x_i + \rho \pi T_i + \kappa_2 x_i T_1  + \xi_{2i}$
* Second stage: $Y_i = \alpha + \beta_1 x_1 + \rho D_i +  \beta_2 x_i D_i+ \eta_i$

Where:

* $T_i = 1(x_i \geq x_0)$ is an indication of the point of discontinuity in predicted income.
* $D_i$ is treatment status that we predict in the first stage.
* $Y_i$ is government approval
* $x_i$ is predicted income, the running variable.

```{r regressions, results='asis'}
df2 <- df2 %>% mutate(T_i = if_else(`Predicted Income` >= 0, 1, 0))

iv1 = ivreg(`Gov. Support` ~ `Predicted Income`*`Received PANES` | `Predicted Income`*T_i , data = df2)

iv_robust <- robust.se(iv1) # robust s.e.
stargazer(iv_robust, ols_fit, type = stargazer_output, 
          dep.var.labels = "Gov. Support", column.labels = c("IV", "OLS"))
```

It is noted that the estimated treatment effect is of about the size indicated by the figures (0.17). This is larger than what the authors report, most likely due to the authors having done some additional data cleaning I've omitted here for brevity and focus on the RDD part. The intercept is larger than 0.5, meaning that more people indicated that they approve of the current president than did not. The interaction term is negative, capturing the negative slope observed to the left in the figures.

```{r}
iv_summary <- summary(iv1, diagnostics = TRUE, vcov = sandwich) # robust s.e. using sandwich
```


In the paper, the authors additionally allow for varying polynomials of order 0, 1, and 2 (columns 1, 2, and 3) on both sides of the threshold (I only report of order 1 here) and also include varying socioeconomic covariates (columns 4, 5, 6) with which the results remain robust. This is precisely what we would expect if the covariates don't also have a discontiuity around the cut off.

Note a few nice properties of the diagnostics, the Wu Hausman statistic is insignificant meaning that it cannot be rejected that OLS and IV estimates are equally consistent. Hence, OLS estimates might have been preferable since they are more efficient. This is also confirmed by the similarity of the point estimates in the table above. Additionally, we have that the first stage is fantastic with a p value on the order of less than computer precision .

```{r, results = 'asis'}
iv_summary
```

## Density around threshold

I begin by showing the density of the running variable around the cutoff to discover potential gaming or self selection around the threshold. If there is gaming, the bar just to the right of the threshold will be "unnaturally" low and the bar to the left will be unaturally high, reflecting that people were bumped to one side because of incentives to do so. If this is the case, the sample will not have been random as some part of the population will be able to self select into treatment. Judging form the graph, it is possible that there is gaming towards the left, where the recipients would just barely qualify for PANES. However, it is not clear cut that this is the case. Fortunately, there are newer methods established to help identify whether there is self selection around the threshold.

Note also that there are few around zero, this likely has to do with the extra survey that ocurred for people close to the threshold.

```{r}
df %>% 
  ggplot(aes(x = ind_reest)) + 
  geom_density(trim = TRUE, alpha = 0.4) +
  geom_histogram(aes(y = ..density..), bins = 50, alpha = 0.5) +
  labs(x = "Predicted income", 
       title = "Discernable difference", subtitle = "However, hard to establish by this graph") +
  theme(
    panel.background = element_rect(fill = "transparent",colour = NA),
    plot.background = element_rect(fill = "transparent",colour = NA))
ggsave("images/density.png", bg = "transparent")


```

I proceed by plotting a local polynomial fit of the density. This plot estimates a local polynomial of the density and is based on the paper by @cattaneo2019simple. The idea is that the polynomial fit doesn't suffer from the same boundary bias as do the kernel estimate above. I used the deafult options for bandwidth as calculated by the package for simplicity.


```{r, message = FALSE, warning = FALSE, results = "hide"}
rdd <- rddensity(X = df$ind_reest)

# Plot
local_poly_fit <- rdplotdensity(rdd, df$ind_reest, plotRange = c(-0.02, 0.02))
```

```{r, include = FALSE}
jpeg("images/local_poly_fit.jpeg")
local_poly_fit$Estplot
dev.off()
```


The graph above puts doubt on whether there was self selection going on as indicated before; the point estimate of the non recipient population is within the confidence interval and there is significant overlap. To statistically test whether there is a problem, I proceed with the McCrary test.


```{r mccrary, message = FALSE, warning = FALSE}
DCdensity(df_mccrary$ind_reest, 0, bin = NULL, bw = NULL, verbose = FALSE, plot = TRUE, ext.out = FALSE, htest = FALSE)
```
```{r,  include = FALSE}
jpeg("images/mccrary.jpeg")
Mccrary_test <- DCdensity(df_mccrary$ind_reest, 0, bin = NULL, bw = NULL, verbose = FALSE, plot = TRUE, ext.out = FALSE, htest = FALSE)
dev.off()
```



The McCrary test (@mccrary) ends up being insignificant at $p =$ `r round(Mccrary_test,3)`. This means we cannot reject the null hypothesis that there is no discontinuity around the density of the running variable, which is good news for the analysis as this indicates there were no gaming going on, pushing people to either side of the threshold, so people on both sides can be said to be relatively similar. This, along with the effect sizes estiamted earlier, leads to a conclusion that the program likely helped the incumbent party.

# Conclusion
It appears as if there is a large causal effect of the government program on government support. This is confirmed by the IV estimates of the RDD and robustness checked by running gaming checks. It could still be the case that a covariate varies around the same threshold and is the driver of the effect. However, this seems unlikely as the running variable is an econometric prediction of reported income that was made up of several variables. That one would be discontinuous around the same value as predicted income needs a pervasive case. 

The main remaining concern is that reported approval of the president is not a good measure of voting intentions. As was noted earlier, Uruguay is not a corrupt country so respondents had no incentive to report support when they don't support the president. Additionally, the parliamentarianism of Uruguay makes it likely that approval of the president is strongly related to voting intentions of the president's party. 

A final note of caution is to interpret the point estimate in this report carefully as the point estimates don't correspond to those of the authors, despite using the same data set. As was noted, this probably boils down to me omitting some data cleaning that the authors may have done to improve the accuracy of reported estimates.

# Rating effects on price on Airbnb

To investigate whether average rating has a causal impact on the log price of a listing, it might be possible to compare listings with similar ratings that are displayed differently to users. The idea is that if ratings are rounded (which they turned out not to be presently, but may have been so in the past when the data was taken), then an institutional feature creates variation that is as good as exogenous since ratings on either side of the rounding cutoff will be similarly rated and not expected to have any other systematic differences.

```{r airbnb, message=FALSE, warning=FALSE}
df <- import("chic_aug_07.csv")
df %>% mutate(PriceX = gsub("[^0-9.]", "", df$price_string),
              PriceX = as.numeric(PriceX),
              cutoff = round((avg_rating-0.25)*2)/2 + 0.25,
              dist_t = avg_rating - cutoff,
              treat = if_else(dist_t == 0, 1, 0)) -> df

df %>% filter(avg_rating > 0) %>% ggplot(aes(x = avg_rating, y = log(PriceX))) + 
  geom_point(alpha = 0.3, color = "turquoise") + geom_smooth(method = "lm", formula = y~x) +
  labs(x = "Average rating", y = "log price", title = "Positive correlation between ratings and price") +
  theme(
        panel.background = element_rect(fill = "transparent",colour = NA),
        plot.background = element_rect(fill = "transparent",colour = NA)
        )
ggsave("images/correlation.png", bg = "transparent")
```
There is a correlation present in the data. Let's investigate whether we can use the techniques intended for this analysis to uncover any causal effect.

```{r airbnb2, message=FALSE, warning=FALSE}
# "Discontinuity plot"

# Rating, log price relationship
df %>% mutate(above = if_else(dist_t >= 0, dist_t, NaN),
                        below = if_else(dist_t < 0, dist_t, NaN)) %>%
  gather(above, below, key = "Treatment", value = "dist_to_cutoff") %>% 
  ggplot(aes(x = dist_to_cutoff, y = log(PriceX), 
             color = Treatment, fill = Treatment)) + 
  geom_smooth(method = lm, color = "black") +
  geom_smooth(method = loess, span = 0.75) +
  geom_point(alpha = 0.3) +
  labs(title = " No discontiuity around threshold", 
       subtitle = "Black lines suggest relationship but are biased",
       x = "Distance to rounding cut off", y = "Log nightly price") +
  theme(
    panel.background = element_rect(fill = "transparent",colour = NA),
    plot.background = element_rect(fill = "transparent",colour = NA))
ggsave("images/airbnb_rd.png", bg = "transparent")
```
```{r}
df %>% mutate(above = if_else(dist_t >= 0, dist_t, NaN),
                        below = if_else(dist_t < 0, dist_t, NaN)) %>%
  filter(abs(dist_t) < 0.15) %>% 
  gather(above, below, key = "Treatment", value = "dist_to_cutoff") %>% 
  ggplot(aes(x = dist_to_cutoff, y = log(PriceX), 
             color = Treatment, fill = Treatment)) + 
  geom_smooth(method = lm) +
  geom_point(alpha = 0.3) +
  labs(title = "Linear estimate with data closer to cutoff", 
       subtitle = "Confirms there is no discontinuity",
       x = "Distance to rounding cut off", y = "Log nightly price") +
  theme(
    panel.background = element_rect(fill = "transparent",colour = NA),
    plot.background = element_rect(fill = "transparent",colour = NA))
ggsave("images/airbnb_rd_close.png", bg = "transparent")
```


The local polynomial fit suggests there is no relatioship. It appears as if there is a relation by judging from the black linear fits. These are likely biased as they are estmated with data quite far from the cutoff. As can be seen in the graph with data less than 0.15 away from the cutoff, the relationship virtually disappears as indicated by the local polynomial.


# Power calculation
I provide a power calculation around the point estimate of half the effect size estimated under the linear distance (jump between the black lines). Note that the black lines might be biased by data far from the threshold and that particularly the one above the cutoff is quite far from the local polynomial fit. 

To obtain the point estimate, it's enough to do estimate a regression using OLS here, as the RDD is "Sharp" (deterministic) meaning the first stage analogy would be precisely 1.

```{r abnb_regressions, results='asis'}
df <- df %>% mutate(T_i = if_else(dist_t >= 0, 1, 0))

ols_abnb <- lm(log(PriceX) ~ dist_t*T_i , data = df)

stargazer(ols_abnb, type = stargazer_output)
```


```{r power}
effect_size_est <- ols_abnb$coefficients[["T_i"]]/2
std_error <- coef(summary(ols_abnb))[3,2]
sample_size <- nrow(df)
sd_est <- std_error*sqrt(sample_size)
power_values <- enframe(seq(0.5, 0.9, 0.001))


req_sample_size <- c()
for(pow in power_values$value){
  req_sample_size <- c(req_sample_size, power.t.test(n = NULL, delta = effect_size_est, sd = sd_est, sig.level = 0.05, power = pow, type = "two.sample", 
                     alternative = "two.sided")$n)
}

req_sample_size2 <- c()
for(pow in power_values$value){
  req_sample_size2 <- c(req_sample_size2, power.t.test(n = NULL, delta = effect_size_est, sd = sd_est, sig.level = 0.01, power = pow, type = "two.sample", 
                     alternative = "two.sided")$n)
}


power_values <- power_values %>% mutate(`p = 5%` = req_sample_size, 
                    `p = 1%` = req_sample_size2,
                    Power = value) %>%
  gather(key = "Significance_level", value = value, `p = 5%`, `p = 1%`)

power_values %>% ggplot(aes(x = value, y = Power, color = Significance_level)) +
  geom_line() + 
  labs(title = "Required sample per desired power",
         x = "Required sample size", y = "Power") +
  theme_minimal()+
  theme(
    panel.background = element_rect(fill = "transparent",colour = NA),
    plot.background = element_rect(fill = "transparent",colour = NA))
ggsave("images/airbnb_power.png", bg = "transparent")
```



# References




```

automatically created on 2019-12-02