---
layout: post
math: true
title:  "Generalized Newton updates for Non-Conjugate Variational Bayes"
date:   2018-02-01
---

In this post I show how to apply generalized Newton updates to achieve fast and easy to implement updates for non-conjugate mean-field variational inference. The rules integrate well with standard coordinate ascent / variational message passing, since they essentially locally approximate non-conjugate messages by conjugate ones. The method is restricted to situations where all expectations can be computed in closed form.

The main steps are

 1. Recognise that coordinate ascent updates are optimization problems
 1. In the locally conjugate case, this optimization can be solved in closed form
 1. Derivative matching a la [Beyond Newton's Method](https://tminka.github.io/papers/minka-newton.pdf) can be used to locally approximate the non-conjugate problem by a conjugate one, the solution of which is the udpate

## Example Problem

I use a Normal-Poisson model as a running example,

$$
\begin{align}
z &\sim \mathcal{N}(\mu_0, \sigma_0^2) \\
x &\sim \text{Poisson}\left(e^z\right)
\end{align}
$$

where $$x$$ is observed and the log-Poisson rate $$z$$ is to be inferred. We assume there are $$n$$ observations with sufficient statistics $$k = \sum_{i=1}^n x_i$$.

The ELBO for this model is

$$
\begin{align}
L[q] &= \mathbb{E}_{z\sim q}\left[\sum_{i=1}^n \log p(x_i \,|\, z) + \log p(z)\right] - \mathbb{E}_{z\sim q}\left[\log q(z)\right] \\
&= \mathbb{E}_{z \sim q}\left[ k z - n e^z - \frac{\left(z - \mu_0\right)^2}{2\sigma_0^2}\right] + \mathbb{H}[q] + \, \text{const}
\end{align}
$$

With a Gaussian variational distribution $$q = \mathcal{N}(m, s^2)$$ we can compute all expectations,

$$
L(m, s^2) = k m - n e^{m}e^{\frac{1}{2}s^2} - \frac{m^2 + s^2 - 2\mu_0 m}{2\sigma_0^2} + \frac{1}{2} \log s^2 + \,\text{const}
$$

When there are no observations ($$n = k = 0$$), the model is conjugate and the maximum be found exactly - unsurprisingly this is just $$m = \mu_0, s^2 = \sigma_0^2$$. The term $$e^{m}e^{\frac{1}{2}s^2}$$ spoils this, however. 

## Generalized Newton's Method

The idea behind the Newton method in minimization is to locally approximate a hard objective by a parabola by matching derivatives. The exact mimimum of the parabola is the next step in an iterative solution.

[Thomas Minka's generalized Newton's method](https://tminka.github.io/papers/minka-newton.pdf) consists in replacing the quadratic term by some other non-linear function that is tailored to the problem at hand.

This idea can be applied to variational inference by collecting all tractable terms and using these for approximating the intractable ones. Concretely, in the above we approximate

$$
-e^{m}e^{\frac{1}{2}s^2} \approx \text{const}\, + a_1 m - \frac{a_2}{2} m^2  - \frac{b_1}{2}s^2 + \frac{b_2}{2} \log s^2
$$

Matching the first two derivatives w.r.t. $$m$$ and $$s^2$$ leads to a triangular linear system with solution

$$
\begin{align}
a_1 &= (m - 1)\lambda \\
a_2 &= \lambda \\
b_1 &= \left(1 + \frac{s^2}{2}\right)\lambda \\
b_2 &= \frac{s^4}{2} \lambda
\end{align}
$$

where $$\lambda = e^m e^{\frac{1}{2}s^2}$$ is the expected Poisson rate.

## Approximate Update Rule

Plugging the approximation into the ELBO leads to a tractable optimization problem with solution

$$
\begin{align}
m_{\text{new}} &= \frac{\frac{\mu_0}{\sigma_0^2} + k +  n\lambda(m - 1)}{\frac{1}{\sigma_0^2} + n \lambda}\\
s^2_{\text{new}} &= \frac{1 + \frac{1}{2} n\lambda s^4}{\frac{1}{\sigma_0^2} + \left(1 + \frac{s^2}{2}\right) n \lambda}\\
\end{align}
$$


Some examples of the convergence are shown below for an empirical rate of $$0.1$$ and various $$n$$. For comparison, the maximum likelihood estimate and corresponding Gamma credible interval is shown as well. Note that discrepancies for small counts are due to the variational Gaussian, which is not a good fit close to the domain boundary.

![examples](/assets/img/normal-poisson-convergence.png)

## Summary

Applying generalized Newton to variational inference leads to simple and efficient algorithms, but is restricted to cases with computable expectations and requires some calculations by hand (even though they could by automated to a large extent). In the future, I may write updates on

 * other tractable examples
 * Comparison to other methods like one of the various black-box VI algorithms
 * Automated derivation of update equations with symbolic algebra software
