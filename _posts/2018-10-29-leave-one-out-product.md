---
layout: post
math: true
title:  "Computing leave-one-out products"
date:   2018-10-29
---

Sometimes it is necessary to compute products of the form

$$
b_i = \prod_{j \neq i} a_i
$$

for an array $$a$$ - in the following I call this the leave-one-out (or loo) product.

A naive numpy one line implementation is

```python
b = np.prod(a) / a
```

but this is numerically unstable when some $$a_i$$ are close to zero. In fact, if one of the $$a_i$$ is numerically indistinguishable from zero this method fails completely.

Luckily, there is a simple recursive method based on message passing ([see e.g. this book, ch. 16](http://www.inference.org.uk/itprnn/book.pdf)) that avoids divisions.

Just rewrite

$$
\begin{align}
L_i &= \prod_{j < i} a_i \\
R_i &= \prod_{j > i} a_i \\
b_i &= L_i R_i
\end{align}
$$

Then both $$L$$ and $$R$$ can be computed recursively like

$$
\begin{align}
L_1 &= 1 \\
L_{i + 1} &= L_i a_i
\end{align}
$$

and similarly for $$R$$.

With proper implementation, this yields $$O(N)$$ run time and $$O(1)$$ memory overhead:

```python
import numpy as np
import numba

@numba.njit
def prod_loo(a):
    """Evaluate the leave-one-out product of a one-dimensional array."""
    LR = 1
    res = np.empty_like(a)
    n = a.size
    # left-to-right pass
    for i in range(n):
        res[i] = LR
        LR *= a[i]
    # right-to-left pass
    LR = 1
    for i in range(n - 1, -1, -1):
        res[i] *= LR
        LR *= a[i]
    return res
```

When compiled with [`numba`](http://numba.pydata.org), this can actually be faster than the naive implementation:

```python
>>> a = 1. + np.random.uniform(-.01, .01, size=100)
>>> %timeit np.prod(a) / a
15 µs ± 215 ns per loop (mean ± std. dev. of 7 runs, 100000 loops each)

>>> %timeit prod_loo(a)
1.91 µs ± 32.7 ns per loop (mean ± std. dev. of 7 runs, 1000000 loops each)
```

Note that underflow and overflow of the result for many large or small inputs is still an issue. However, the basic logic carries over to the log-domain - doing sums when some terms can be $$-\infty$$.

