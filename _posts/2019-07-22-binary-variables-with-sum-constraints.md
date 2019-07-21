---
layout: post
math: true
title: "Binary Vectors with Sum Constraints"
date: 2019-07-22
---


# Binary vectors with sum constraints

In this post we look into the statistics of binary variables with constraints on their sum, but otherwise independent. A well-known special case is that of categorical variables, where we constrain the sum to one. How can this be generalized to more general constraints?

__Note:__ While the solution below works in principle, it is not numerically stable. In the next part I discuss how to make it practical.

## The Problem

Specifically, consider a binary vector $$\mathbf{x} = (x_1, \ldots, x_N)$$. We use the short hand $$\nu(\mathbf{x}) = \sum_{i=1}^N x_i$$, so that the constraint can be written as $$f_{\nu(\mathbf{x})}$$ for some non-negative function $$f_n, 0 \leq n \leq N$$. The partition function is

$$
Z(\mu) = \sum_{\mathbf{x}} e^{\sum_i \mu_i x_i} \, f_{\nu(\mathbf{x})}
$$

We want to evaluate the marginals $$p_i = \mathbf{E}[x_i]$$. The presence of the constraint does not affect the validity of the standard formula

$$
p_i = \partial_{\mu_i} \log Z(\mu) = \frac{\partial_{\mu_i}Z(\mu)}{Z(\mu)}
$$

since $$Z(\mu)$$ is still an exponential family. The problem boils down to computing $$Z(\mu)$$ and its gradient.

### Basic Example

For the categorical case, $$f_n = \delta_{n, 1}$$:

$$
\begin{align}
Z   &= \sum_{\mathbf{x}} e^{\sum_i \mu_i x_i} \, \delta_{\nu(\mathbf{x}), 1} \\
    &= \sum_{i=1}^N e^{\mu_i}
\end{align}
$$

leading to the marginals

$$
\frac{\partial_{\mu_i}Z}{Z} = \frac{e^{\mu_i}}{\sum_i e^{\mu_i}}
$$

We could use direct summation due to the simplicity of the problem. For more general constraints, however, direct summation becomes very expensive - e.g. for $$f_n = \delta_{n, k}$$ the number of terms in the expression for $$Z$$ is $$\begin{pmatrix}N \\ k\end{pmatrix} = O\left(\min\left(N^k, N^{N - k}\right)\right)$$.

## General Solution

We can use the special structure of the constraint, and introduce the Fourier transform

$$
\begin{align}
\tilde{f}(\lambda) &= \sum_{n=0}^N f_n e^{-i\lambda n} \\
f_n &= \int_{-\pi}^{\pi} \tilde{f}(\lambda) \, e^{i\lambda n}\, \frac{d\lambda}{2\pi}
\end{align}
$$

Now see what happens to the partition function:

$$
\begin{align}
Z(\mu)
    &= \sum_{\mathbf{x}} e^{\sum_j \mu_j x_j}
       \int_{-\pi}^{\pi} \tilde{f}(\lambda) e^{i\lambda \nu(\mathbf{x})} \frac{d\lambda}{2\pi} \\
    &= \int_{-\pi}^{\pi} \frac{d\lambda}{2\pi} \, \tilde{f}(\lambda)
       \sum_{\mathbf{x}} e^{\sum_j \left(\mu_j + i\lambda\right) x_j} \\
    &= \int_{-\pi}^{\pi} \frac{d\lambda}{2\pi} \, \tilde{f}(\lambda) \prod_j \left[1 + e^{\mu_j + i\lambda}\right]
\end{align}
$$

The summation involving up to $$2^N$$ terms can be evaluated in closed form and we are left with a one-dimensional integral. Similarly, for the gradient we obtain

$$
\partial_{\mu_i} Z(\mu) = \int_{-\pi}^{\pi} \frac{d\lambda}{2\pi} \, \tilde{f}(\lambda) \, e^{\mu_i + i\lambda}\prod_{j\neq i} \left[1 + e^{\mu_j + i\lambda}\right]
$$

### Basic Example Revisited

To understand how we recover the direct results above, all that is needed is

$$
\int_{-\pi}^{\pi}\frac{d\lambda}{2\pi} e^{i n\lambda} = \delta_{n,0}
$$

We only consider the partition function of categorical variables, where $$\tilde{f}(\lambda) = e^{-i\lambda}$$:

$$
\begin{align}
Z(\mu)
    &= \int_{-\pi}^{\pi}\frac{d\lambda}{2\pi} e^{-i\lambda}
       \prod_{j=1}^N \left[1 + e^{\mu_j + i \lambda}\right] \\
    &= \int_{-\pi}^{\pi}\frac{d\lambda}{2\pi} e^{-i\lambda}
       \left[1 + e^{i\lambda} \sum_{j=1}^N e^{\mu_j} + e^{2i\lambda}\left(\cdots\right) + \ldots\right] \\
    &= \sum_{j=1}^N e^{\mu_j}
\end{align}
$$

Here we can see the source of the aforementioned instability: While all the terms in the ellipsis integrate to zero, they can shadow the result due to floating point precision in practice.

### Complexity

The complexity of evaluating the marginals in this way is $$O(N^2)$$:

 * All required products can be evaluated simultaneously in $$O(N)$$ using the sum-product algorithm (implementation discussed [here]({% post_url 2018-10-29-leave-one-out-product %}))
 * Denoting $$z = e^{i\lambda}$$, only powers of $$z^k$$ with $$-N \leq k \leq N$$ occur. This implies that the trapezoidal integration rule with $N + 1$ points solves the integral exactly (see e.g. [here](https://epubs.siam.org/doi/pdf/10.1137/130932132))
 * Correspondingly, we can use the FFT to compute the required values of $$\tilde{f}(\lambda)$$ once in $$O(N\log_2 N)$$

The last two points should not be surprising because we could have used the discrete Fourier transform instead of the continuous one.
