---
title: Confidence intervals for non-linear fitting
layout: post
nav: blog
comments: true
---

Finding confidence intervals (CI) for fitting parameters is relatively easy when performing linear fits and there's a lot of material covering that. However, things are a bit more tricky when fitting data with a non-linear function. Here, I will look at how to estimate parameter CI in such a case and compare two approaches.

## Task

We have some data which we want to fit a model to. It looks like this:
![](/img/ftest/raw_data.png)

You wouldn't say it from looking at the data, but let's assume we have some prior knowledge regarding the process generating the data and it can be described by a simple exponential function: 
\\[1-\exp(-t/\tau)\\]
(The data was generated with this function by setting \\(\tau\\) to`25 ms` and I've made the task a bit more complicated by removing all datapoints earlier than 30 seconds.)

We're interested in what is the value of \\(\tau\\)  and how accurately do know this value (ie., the confidence interval). 

## First 

The quick and easy way to fit the data with our function is using the `curve_fit` function in [`scipy.optimize`][1]. For this particular dataset and function it will estimate \\(\tau\\) to be `27.34 ms`. The optimizer also provides estimates for the variance of the parameter estimate \\(\sigma\_{\tau}\^2\\).

Using this we can easily calculate the CI for \\(\tau\\) at confidence level \\(\alpha\\):
$$\tau \pm t\_{\alpha/2, n-1}\cdot\sqrt{\sigma\_{\tau}\^2}$$
where \\(t\_{\alpha/2, n-1}\\) is the critical value of the Student's t distribution at at \\(\alpha/2\\) and n-1 degrees of freedom.
Using this we get for 95% and 99% CI:
$$\mathrm{CI}\_{\tau,95\%} = 27.34 \pm 2.05 \cdot 7.02 = (12.9, 41.7)$$
and
$$\mathrm{CI}\_{\tau,99\%} = 27.34 \pm 2.76 \cdot 7.02 = (7.99, 46.9 )$$

The CI are quite large but considering the quality of data it's not surprising. Graphically the fit and CI look like this (light shaded area is for 99% CI and dark for 95% CI)

![](/img/ftest/asym_cis.png)

The CI obtained with the method above are approximate because the variance provided by the non-linear fitting routine is asymptotic. Especially in the case like this, where the CI are large, it might be a good idea to check whether or not the approximation is good or not.

## Second

For a more accurate estimation of the CI we use the F-test. If we have two fits for the same data, we could use the F-test to determine whether or not the fits are significantly different (at some confidence level). Equivalently, we can fix a confidence level and a one of the fits, then vary the other fit parameter until their F-ratio reaches the critical value for the given confidence level. This will give the CI for the fit parameter.

The general formula to calculate [F-ratio][2] for two fits is:
\\[F=\frac{\frac{\mathrm{RSS}\_0 - \mathrm{RSS}\_1}{\mathrm{RSS}\_1}}{\frac{df\_0-df\_1}{df\_1}}\\]

Where *RSS* is the the residual sum of squares and *df* the degrees of freedom. If *y* are the original datapoints and *f(t)* the estimates obtained from the fit then:
\\[\mathrm{RSS} = \sum\_i (y\_i-f(t\_i))\^2\\]

In this case *RSS<sub>1</sub>* is the best fit with lowest residual sum of squares ( *RSS<sub>best</sub>* ) obtained from the optimizer ( *&tau;=27.34 ms* ) and *RSS<sub>0</sub>* is the target sum of squares for *&tau;* value we are looking for ( *RSS<sub>target</sub>* ). The degrees of freedom are: \\(df\_1 = n-p\\) and \\(df\_0 = n\\), where *n* is the number of datapoints and *p* the number of parameters (1 in this case). Plugging all this into the F-ratio formula:
\\[F = \frac{\frac{\mathrm{RSS}\_0 - \mathrm{RSS}\_1}{\mathrm{RSS}\_1}}{\frac{df\_0-df\_1}{df\_1}} = 
\frac{\mathrm{RSS}\_{target}/\mathrm{RSS}\_{best}-1}{p/(n-p)}
\\]
When looking for CI for a given confidence level we effectively fix the critical F value to the *&alpha;* level and degrees of freedom (the [F distribution][3] has two degrees of freedom). We are looking to find a *&tau;* value for which *RSS* reaches:
\\[\mathrm{RSS}_{target,\alpha} = \mathrm{RSS}\_{best}\cdot\left(F\_{\alpha, p, n-p}\cdot\frac{p}{n-p}+1\right)\\]

For the data in the example above *RSS<sub>best</sub> = 3.39*, *p=1*, *n-p = 29* :
\\[\mathrm{RSS}_{target,\alpha} = \mathrm{RSS}\_{best}\cdot\left(F\_{\alpha, p, n-p}\cdot\frac{p}{n-p}+1\right)\\]

At confidence level 95%, *F<sub>.95,1,29</sub>=4.18* and:
\\[\mathrm{RSS}\_{target,95\%} = 3.39 \cdot \left(4.18 \cdot\frac{1}{29}+1\right)=3.88\\]
and at confidence level 99% with *F<sub>.99,1,29</sub>=7.60*:
\\[\mathrm{RSS}\_{target,99\%} = 3.39 \cdot \left(4.18 \cdot\frac{1}{29}+1\right)=4.28\\]

We can calculate and plot the *RSS* for a range of *&tau;* values and find where the target *RSS* levels intersect with the curve:
![](/img/ftest/res_target.png)

From the intersection points (green dots for 95% and orange for 99%) we get the CI for *&tau;*:

$$\mathrm{CI}\_{\tau,95\%} = (10.86, 43.26)$$
and
$$\mathrm{CI}\_{\tau,99\%} = (0, 49.67 )$$

Not only are the CI larger than obtained from with the asymptotic method, but as the target *RSS* level only intersects with the *RSS* curve at one point the lower bound at 99% confidence level doesn't even exist. Because negative *&tau;* values don't make sense we fix it to be 0, but essentially our data is inconclusive with regards to the lower CI bound at 99% confidence level.
![](/img/ftest/ftest_cis.png)

## Comparison
Comparing the CI obtained with the two approaches it's clear that the asymptotic method provides an *over*-confident CI. In the case of 99% CI it yields a lower bound which does not actually exist! 
![](/img/ftest/ci_compare.png)

In the case when the data has low noise and is not quite as ambiguous as above:
![](/img/ftest/raw_data_close.png)
both methods yield comparable results:
![](/img/ftest/ci_compare_close.png)
Note that the vertical scale on the right panel is almost 8 times smaller than in the more noisy case above.

## Summary



[1]:http://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.curve_fit.html
[2]:http://en.wikipedia.org/wiki/F-test#Regression_problems
[3]:http://en.wikipedia.org/wiki/F-distribution

