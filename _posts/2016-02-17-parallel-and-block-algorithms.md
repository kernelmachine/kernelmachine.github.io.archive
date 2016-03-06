---
layout: post
title: Parallel and Block algorithms in Rust
comments: true
---


### Block operations

Standard matrix multiplication between two $$ n \times n $$ matrices $$A$$ and $$B$$ involves $$2n^3$$ operations:

{% highlight python %}
for i in range(0, n):
  for j in range(0,n):
    for k in range(0,n):
      C[i,j] = C[i,j] + A[i,k]B[k,j]
{% endhighlight %}


However, when you make a simple change to the order of operations in the loop, you can achieve much faster computations.


{% highlight python %}
for i in range(0, n):
  for j in range(0,n):
    for k in range(0,n):
      C[i,j] = C[i,j] + A[i,k]B[k,j]
{% endhighlight %}

Performance benefits are rooted in optimized cache performance. Cache locality (temporal/spatial), memory efficiency vs computing
efficiency.


###  Blocks and Parallelization
Gaussian etc


### Parallelization in Rust

pub fn parallel_matrix_multiplication(M : Matrix){
  M.split_
}
