---
layout: post
math: true
title: "Expectation Propagation for Probabilistic Bipartite Matching"
date: 2019-10-10
---

__Given two sets of objects and a matrix of probabilities for matching them, how can we approximate the matching marginals?__


In this post I sketch an iterative algorithm for probabilistic bipartite matching. A weighted bipartite matching instance consists of a bipartite graph with a weight associated to each edge. A valid matching assigns exactly one vertex from one partition to one in the other partition. In a probabilistic setting, the weights are log-probabilities, and we do not ask for the maximum matching, but for the distribution of matchings.

More specifically, we denote the partitions by $$V, W$$ with vertices $$v_i, i = 1,\ldots, N$$ and $$w_j, j = 1,\ldots,M$$. The edges are represented by a matrix of potentials $$\mu$$, and the matching variable by $$z$$, with $$z_{ij} = 1$$ if $$(v_i, w_j)$$ is in the matching. For now, we focus on perfect matchings $$M = N$$ and all vertices matched. The corresponding constraints are

$$
\begin{align}
\sum_j z_{ij} &= 1 &\text{Rows sum to one}\\
\sum_i z_{ij} &= 1 &\text{Columns sum to one}
\end{align}
$$

Denote the indicator functions for the row (column) constraints by $$\nu_r(z)$$ and $$\nu_c(z)$$, respectively. Then the probability is

$$
p(z) = \frac{1}{Z} e^{\sum_{ij}\mu_{ij} z_{ij}} \,\nu_r(z) \nu_c(z)
$$

where $$Z = \sum_z p(z)$$ is the partition function. Evaluating the partition function is in [#P][sharp-p], but there are tractable partial problems that allow for efficient simplification.

Namely, if $$\nu_c(z)$$ is dropped we are left with independent Bernoullis with a sum-to-one constraint on the rows - i.e. just a bunch of independent categorical variables. The same holds for dropping $$\nu_r(z)$$. This so-called [measure factorization][jordan-combinatorial] allows to apply [Expectation Propagation][bishop-ep] (EP) to this problem.

Recall that in EP each intractable factor is approximated by a simpler distribution. Here we approximate all factors by independent Bernoulli distributions. We use $$\rho$$ and $$\gamma$$ for the approximations of $$\nu_r$$ and $$\nu_c$$, so that the approximated posterior has marginals

$$
p_{ij} = \frac{1}{1 + e^{-\left(\mu_{ij} + \rho_{ij} + \gamma_{ij}\right)}}
$$

To update $$\rho$$, we need to compute marginals under the distribution

$$
q_r(z) \propto \nu_r(z) e^{\sum_{ij}\mu_{ij} + \gamma_{ij}}
$$

which is simply

$$
\begin{align}
\pi_{ij} &=\mathbb{E}_{z\sim q_r}\left[z_{ij} \right] \\
&= \frac{e^{\mu_{ij} + \gamma_{ij}}}{\sum_{j} e^{\mu_{ij} + \gamma_{ij}}}
\end{align}
$$

In order to update the approximation, set

$$
\rho = \log \frac{\pi}{1 - \pi} - \left(\mu + \gamma\right)
$$

The updates for $$\gamma$$ works in the same way, doing both updates is one EP iteration.


# Links

1. [Complexity of counting perfect matchings (Wikipedia)][sharp-p]
2. [M Jordan, A Bouchard-côté: Variational Inference over Combinatorial Spaces][jordan-combinatorial] contains a more sophisticated approach to similar problems.
3. C Bishop, Pattern Recognition and Machine Learning (Ch. 10.7). [Official free PDF][bishop-ep]

[sharp-p]:https://en.wikipedia.org/wiki/Sharp-P-completeness_of_01-permanent#Significance
[jordan-combinatorial]:https://papers.nips.cc/paper/4036-variational-inference-over-combinatorial-spaces
[bishop-ep]:https://www.microsoft.com/en-us/research/people/cmbishop/prml-book/
