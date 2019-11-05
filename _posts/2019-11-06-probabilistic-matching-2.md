---
layout: post
math: true
title: "Entropy Approximations for Probabilistic Bipartite Matching"
date: 2019-11-06
---

This is a continuation of [this post]({% post_url 2019-10-10-probabilistic-matching %}). It is somewhat complementary, in that instead of starting from update equations I consider a general family of approximations based on entropy approximations. Expectation Propagation is just one such approximation and can be improved upon by considering other members of the same family. This post uses material from [Wainwright and Jordan, Graphical Models, Exponential Families, and Variational Inference] throughout. The relevant sections are listed at the bottom.

## Bipartite Matching as Exponential Family

We consider matching of two sets of objects $$U$$ and $$V$$, given a matrix of match probabilities. The goal is to infer the posterior distribution over matches. We use a binary variable $$z_{ij} = 1$$ if $$u_i$$ and $$v_j$$ are matched.

We consider perfect matchings only and assume that $$U$$ and $$V$$ have the same size $$N$$. In this case the matrix $$z_{ij}$$ is square. There are only $$N!$$ possible matchings, requiring constraints on the variables like

$$
\begin{align}
\sum_{j} z_{ij} &= 1 & \text{Rows sum to one}\\
\sum_{i} z_{ij} &= 1& \text{Columns sum to one}
\end{align}
$$

We write $$\nu_r(z), \nu_c(z)$$ for the row and column constraints and obtain the partition function

$$
Z(\theta) = \sum_{z} \nu_r(z)\nu_c(z) \; e^{\sum_{ij} \mu_{ij}\theta_{ij}}
$$

This defines an exponential family with natural parameters $$\theta$$, the logits of the match probabilities. Note that this has one parameter too many because shifting all $$\theta_{ij}$$ by the same fixed amount doesn't alter the distribution. The sufficient statistics are simply $$z_{ij}$$. We write

$$
\mu_{ij} = \mathbb{E}\left[z_{ij} \right]
$$

for the expected sufficient statistics. By linearity the $$\mu$$ inherit the constraints from the $$z$$, e.g. $$\sum_j \mu_{ij} = 1$$. We write $$M$$ for the space of feasible mean parameters including the constraints $$0 \leq \mu_{ij}\leq 1$$ and the sum constraints.

## A Family of Entropy Approximations

We can make use of the exponential family form and employ a general variational form of the log partition function

$$
A(\theta) = \max_{\mu \in M} \sum_{ij} \theta_{ij}\mu_{ij} - A^\ast(\mu)
$$

where $$A^\ast(\mu)$$ is the convex dual to the log partition function $$A(\theta)$$. It is also the negative entropy w.r.t. the base measure. This representation allows to devise approximations based on either approximations of $$M$$ or the entropy. Since $$M$$ is tractable, we focus on approximating the entropy. More specifically, we use separable entropies of the form

$$
\mathbb{H}(\mu) = \sum_{ij} H(\mu_{ij}).
$$

This leads to the optimization problem

$$
A(\theta) \approx \max_{\mu \in M} \sum_{ij}\left[\theta_{ij}\mu_{ij} + H(\mu_{ij})\right]
$$

## Problems with Expectation Propagation

For the EP equation derived in [this post]({% post_url 2019-10-10-probabilistic-matching %}) we used independent Bernoullis as the approximation. The corresponding entropy approximation consists of the entropy of the approximation, and adds the difference of the entropies of each approximated term and the Bernoulli entropy.

The Bernoulli entropy is

$$
H_B(\mu) = -\mu\log\mu - (1 - \mu)\log(1 - \mu)
$$

The approximated terms are the sum constraints. Since a Bernoulli vector with sum constraint is a categorical variable, the entropy of each approximated term (per variable) is

$$
H_C(\mu) = -\mu\log\mu
$$

The full entropy is

$$
\begin{align}
H(\mu) &= H_B(\mu) + \underbrace{H_C(\mu) - H_B(\mu)}_{\text{row constraints}} + \underbrace{H_C(\mu) - H_B(\mu)}_{\text{column constraints}} \\
&= -\mu\log\mu + (1-\mu)\log(1 - \mu)
\end{align}
$$

Note the plus sign of the last term. This causes the entropy to be anti-symmetric in $$\mu$$ as well as non-convex. This leads to undesirable behavior that can be illustrated with a toy problem:

### Two-by-Two Matching Problem

The two-by-two case is the smallest non-trivial matching problem. Satisfying all sum constraints reduces the number of parameters to a single number $$\nu$$ such that

$$
\mu = \begin{pmatrix} \nu & 1 - \nu \\ 1 - \nu & \nu \end{pmatrix}
$$

But since the EP entropy is anti-symmetric, it is clear that

$$
\sum_{ij} H(\mu_{ij}) = 0
$$

regardless of the value of $$\nu$$. As a result, the EP equations move to the maximum likelihood solution regardless of the parameters $$\theta$$ instead of approximating the marginals. Indeed, trying to understand this behaviour for the EP iteration during testing was the motivation for this post. While such small systems are hardly of interest for approximations, they tend to occur approximately as sub-units of larger problems where they lead to overconfidence.

In the next section we modify the entropy approximation so as to obtain a more conservative approximation.

## Improved Approximation

The simplicity of the EP equations derive from the separability of the entropy, while its shortcomings originate in its specific form. We can derive other approximations by using a different functional form of $$H(\mu)$$.

In particular, we can set

$$
H(\mu) = -\mu\log\mu
$$

which has the benefit of being concave. Since the constraints are linear, the optimization problem is convex:

$$
\begin{align}
\log A(\theta) &= \max_{\mu \in M} \sum_{ij}\left[\theta_{ij}\mu_{ij} - \mu_{ij}\log\mu_{ij}\right]
\end{align}
$$

We add Lagrange multipliers $$\lambda_{r,i}, \lambda_{c,j}$$ in order to enforce the sum constraints, while enforcing the domain constraint $$0 \leq \mu_{ij} \leq 1$$ explicitly.

The gradient of the Lagrangian $$L(\mu, \lambda_r, \lambda_c)$$ w.r.t. the means is

$$
\begin{align}
\partial_{\mu_{ij}} L &= \theta + \lambda_{r,i} + \lambda_{c,j} - \log\mu + \text{const}
\end{align}
$$

This is readily solved for $$\mu_{ij}$$ to obtain

$$
\mu \propto e^{\theta_{ij} + \lambda_{r,i} + \lambda_{c,j}}
$$

Plugging this into the gradients w.r.t. the Lagrange multipliers yields e.g. for $$\lambda_{r,i}$$ that

$$
\partial_{\lambda_{r,i}}L = -1 + e^{\lambda_{r,i}}\sum_j e^{\theta_{ij} + \lambda_{c,j}}
$$

These equations can be plugged into any gradient-based solver. In the following we derive an alternative iterative algorithm.

We obtain a coordinate ascent algorithm by solving for $$\lambda_{r,i}$$,

$$
\lambda_{r,i} = \frac{1}{\sum_j e^{\theta_{ij} + \lambda_{c,j}}}
$$

Since all rows are independent of each other (and the same for the columns), the updates can be implemented by initializing

$$
\mu_{ij} = e^{\theta_{ij}}
$$

and iterating

$$
\begin{align}
\mu_{ij} &\leftarrow \frac{\mu_{ij}}{\sum_j\mu_{ij}} \\
\mu_{ij} &\leftarrow \frac{\mu_{ij}}{\sum_i\mu_{ij}}
\end{align}
$$

that is, we alternate normalizing the rows and columns. By the convexity of the overall optimization problem the iteration converges to a unique optimum. Moreover, empirically the iterations do not suffer from the overconfidence of the EP equations.

### Two-by-Two Matching Revisited

We use the same parametrization as above. In terms of the parameter $$\nu$$, the optimization problem is

$$
A(\theta) = \max_\nu \left(\theta_{11} + \theta_{22}\right)\nu  + \left(\theta_{12} - \theta_{21}\right)\,(1 - \nu) + 2 H_B(\nu)
$$

For comparison, since there are only two allowed configurations, the exact solution is equivalent to a Bernoulli variable. Hence the approximate entropy is off by a factor of two. Note that the entropy term is larger than in the exact solution, so that the approximation yields solutions that are too diffuse rather than too confident.

### Adjusting the Temperature

Motivated by the two-by-two example, we can adjust the approximate entropy by scaling it like

$$
H(\mu) \rightarrow T H(\mu)
$$

where the scale $$T$$ plays the role of temperature. Heuristically, we can fix the temperature by demanding that the maximal attainable entropy is the same as for the exact entropy (the minimal entropy is zero). In the case at hand, this is $$\log N!$$, while for the approximation it is $$N\log N$$.

This fixes the temperature to

$$
T = \frac{\log N!}{N\log N}
$$

For $$N = 2$$ this recovers the exact solution.

## Conclusions

In this post I outlined a simple family of approximations for probabilistic (perfect) bipartite matching. While the focus on perfect matching may seem limiting, extensions to more general settings (e.g. $$N$$-by-$$M$$ matching) are straightforward.

It may be interesting to consider more general adjustments than the simple temperature scaling above. In principle, the entropy can be replaced with any concave function so that there is a lot of room for improvement.

## Links

1. [M. J. Wainwright, M. I. Jordan: Graphical Models, Exponential Families, and Variational Inference (2008)][wainwright-jordan]
 * The variational formulation of the log partition function is presented in Section 3.6
 * The derivation of EP from an entropy approximation is in Section 4.3

[wainwright-jordan]:https://people.eecs.berkeley.edu/~wainwrig/Papers/WaiJor08_FTML.pdf
