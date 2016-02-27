---
layout: post
title: Parallel and Block algorithms in Rust
comments: true
---

Intro

Standard matrix multiplication between two $$ n \times n $$ matrices $$A$$ and $$B$$ involves $$2n^3$$ operations:

{% highlight python %}
for i in range(0, n):
  for j in range(0,n):
    for k in range(0,n):
      C[i,j] = C[i,j] + A[i,k]B[k,j]
{% endhighlight %}

Block operations

Parallel Array operations in Python

The most interesting parallelization library in Python comes from Dask[^1].

Quick description of dask and how it works.

Parallelization in Rust

Rust is cooler! Native parallelization framework possible via Rayon + NDArray.
