---
layout: post
title: Matrix Multiplication in Rust (Part 1)
comments: true
---

*Thanks to staticassert for feedback, and bluss for conversations. Source code can be found [here](http://github.com/pegasos1/rsmat).*

*This post is the first of a series.*

Basic Linear Algebra Subprograms (BLAS) are the routines that underpin most high-level computational libraries today. Written in C and FORTRAN, they are acutely optimized for speed, and take advantage of special hardware features to achieve unparalleled performance in matrix computations.

It's an early time for Rust, but we're already seeing strides (pun intended) in the numeric computing space.[^1][^2] I was recently inspired to investigate how to make fast matrix computations in Rust, using ndarray-rblas compiled with OpenBLAS[^3] as the target benchmark. This blog series is a documentation of those explorations.

In this post, I'll detail, on a high-level, the process of making fast matrix computations.

The analyses in this post will involve the use of rust-ndarray[^1] for matrix computations. All the crate-specific methods I use are documented [here](http://bluss.github.io/rust-ndarray/master/ndarray/index.html). I'm on a basic Linux laptop: 4-core, i5 CPU with 8 GB RAM.


## Naive dot product
------------

A fundamental operation in numeric computing is the dot product between matrices $$A = [a_1 ,..., a_n]$$ and $$B = [b_1,...,b_n]$$:

$$ dot(A,B) = a_1b_1 + a_2b_2 + ... + a_nb_n $$

It's a simple operation and a great way to dig into performance optimization. So let's start here!

We'll use the following type aliases:

```
pub type VectorView<'a, A> = ArrayView<'a, A, Ix>;
pub type MatView<'a, A> = ArrayView<'a, A, (Ix, Ix)>;
pub type MatViewMut<'a, A> = ArrayViewMut<'a, A, (Ix, Ix)>;
```

ArrayView(Mut)s are (mutable) references to a matrix, or part of it. They're good objects to use if you want to  work with modular subviews of a matrix.

We begin with a naive implementation of the dot product, in which we fold a multiplicative sum of values in vectors `a` and `b`.

```
pub fn vector_dot(a : VectorView<f64>,b:  VectorView<f64>) -> f64 {
        (0..a.len()).fold(0.0, |x, y| x + a.get(y).unwrap() * b.get(y).unwrap() )
}
```

We then define a matrix dot product like so:

```
pub fn matrix_dot( a : &MatView<f64>, b: &MatView<f64>, c : &mut MatViewMut<f64>){
    let ((m,k1),(k2,n)) = (a.dim(),b.dim());
    assert_eq!(k1,k2);
    for ix in 0..m{
        for jx in 0..n{
            let a_row = a.row(ix);
            let b_col = b.column(jx);
            let mut value = c.get((ix,jx)).unwrap();
            *value += vector_dot(a_row, b_col);
        }
    }
}
```

In the code above, we first check whether the inner and outer dimensions of `a` and `b` (respectively) are the same. Then we loop through the matrices' rows and columns, applying our `vector_dot` function and updating a separate matrix `c` with the corresponding output.

Here are the benchmarks, compared to OpenBLAS, on multiplying $$128 \times 100$$ and $$100 \times 128$$ matrices:

```
test bench_dot_dumb        ... bench:   2,843,823 ns/iter (+/- 110,881)

test bench_dot_openblas    ... bench:   201,170 ns/iter (+/- 18,666)
```

Wow, that sucks. We're at least 10x slower than OpenBLAS. But the above implementation is called naive for a reason; we can do a lot better.


## Unsafe indexing
-------------------------
`get()` and `get_mut()`  checks matrix dimensions before returning the indexed value. This makes for safe code, since ndarray will throw an error if the supplied index falls outside of the matrix dimensions (hence the `unwrap`). However, index checking wastes time, and we're trying to make things as fast as possible.

The first thing we could do is index the matrix `c` with unsafe methods `uget()` and `uget_mut()`. These don't check the bounds of the matrix when indexing, and will save us time.

```
pub fn vector_dot_unsafe(a : VectorView<f64>,b:  VectorView<f64>) -> f64 {
    unsafe{
        (0..a.len()).fold(0.0, |x, y| x + a.uget(y) * b.uget(y))
    }

}
```

```
pub fn matrix_dot_unsafe( a : &MatView<f64>, b: &MatView<f64>,
                        c : &mut MatViewMut<f64>){
    let ((m,k1),(k2,n)) = (a.dim(),b.dim());
    assert_eq!(k1,k2);
    for ix in 0..m{
        for jx in 0..n{
            let a_row = a.row(ix);
            let b_col = b.column(jx);
            unsafe{
                let mut value = c.uget_mut((ix,jx));
                *value += vector_dot_unsafe( a_row, b_col);
            }
        }
    }
}
```

We achieve about a 2x speedup with this change.

```
test bench_dot_dumb        ... bench:   1,518,253 ns/iter (+/- 24,523)

test bench_dot_openblas    ... bench:   201,170 ns/iter (+/- 18,666)
```

## Improve matrix multiplication
-------------------------------
The second thing we can do is employ some black magic to improve our matrix multiplication.

The matrixmultiply crate[^4] computes the dot product by decomposing the full computation into a layered set of mini subproblems that can be efficiently packed into the cache. For an in-depth review of the algorithm and its implementation, I'll refer you to another post on the subject [^5].

By replacing the naive dot product with this smarter version, natively supported in ndarray via the method `dot`, we achieve significant speed up:


```
pub fn matrix_dot( left : &MatView<f64>, right: &MatView<f64>,
                  init : &mut MatViewMut<f64>){
    let dot_prod = left.dot(right);
    for ix in 0..m{
        for jx in 0..n{
            unsafe{
            let mut value = init.uget_mut((ix,jx));
            let res = dot_prod.uget_mut((ix,jx));
            *value += res;
          }
        }
    }
}
```

```
test bench_dot_matrixmultiply      ... bench:     370,281 ns/iter (+/- 14,179)

test bench_dot_openblas        ... bench:   201,170 ns/iter (+/- 18,666)
```

Nice! We've increased the speed almost 4x. Now let's try to get our algorithm even closer to OpenBLAS performance.


## Avoid explicit bounds checking
-------------------------------------
Let's revisit the indexing issue in the loop. We gained some significant speedups by making our code unsafe, but this isn't ideal. Let's see if we can get speed *and* safety.

The trick here is that matrix elements in ndarray implement `Iterator`, and *iteration naturally lends itself to bounds checking elimination optimization*. It's implicitly safe to iterate over the matrix. We can assign the values of our dot product to `c` safely without having to make sure we're within the matrix dimensions.

With this in mind, we can simplify the loop to a one-liner that involves the iterator method `zip`:

```
c.zip_mut_with(&dot_prod, |x, y| *x = *y)
```

This results in a smarter, faster matrix multiplication below:

```
pub fn matrix_dot_safe(a: &MatView<f64>, b: &MatView<f64>,
  c: &mut MatViewMut<f64>) {
    let ((_,k1),(k2,_)) = (a.dim(),b.dim());
    assert_eq!(k1,k2);
    let dot_prod = a.dot(b);
    c.zip_mut_with(&dot_prod, |x, y| *x = *y)
}
```

Because ndarray matrices implement `Iterator`, we can use array methods to achieve safety without sacrificing speed. And we get simplicity for free.


```bash
test bench_dot_safe            ... bench:     346,677 ns/iter (+/- 13,246)

test bench_dot_openblas        ... bench:   201,170 ns/iter (+/- 18,666)
```

Why are we still assigning the dot product of  `a` and `b` to `c`? Well, we're just thinking one step ahead - towards parallelization.

## Parallelization
-------------------

Another optimization we can make involves multi-threading. Rust is great for writing concurrent programs because its memory management and type system free the developer from dealing with data races.[^6]

There are quite a few crates that assist the developer in making multi-threaded applications, but the one I'll focus on here is Rayon[^7].

The core of Rayon's API is a work-stealing thread pool. Worker threads pop tasks from a queue, and additional threads are spawned as needed to "steal" future work from busy threads. The really interesting thing about Rayon is that parallelism is not *guaranteed*; instead, it's dependent on whether idle cores are available. It's great to release the developer from thinking about when to use concurrency, and let the API handle it. For an explanation on how this works, I'll refer you to a series of posts here[^8].

Rayon works best with divide-and-conquer algorithms, which happens to be an optimal technique for matrix multiplication.

<img src="http://www.catonmat.net/blog/wp-content/uploads/2009/12/matrix-multiplication-blocks.gif" width="700">

We call `join` on recursively divided matrices, spawning threads to perform multiplication on blocks when we reach some minimum dimension, called `BLOCKSIZE`. The next section will detail the effects of the `BLOCKSIZE` parameter on performance, but for now we arbitrarily set it to 64.

```
pub const BLOCKSIZE: usize = 64;

pub fn matrix_dot_rayon(a: &MatView<f64>, b: &MatView<f64>,
                        c: &mut MatViewMut<f64>) {

    let (m, k1) = a.dim();
    let (k2, n) = b.dim();
    assert_eq!(k1, k2);

    if m <= BLOCKSIZE && n <= BLOCKSIZE {
        matrix_dot_safe(a, b, c);
        return;
    } else if m > BLOCKSIZE {
        let mid = m / 2;
        let (a_0, a_1) = a.split_at(Axis(0), mid);
        let (mut c_0,
            mut c_1) = c.view_mut().split_at(Axis(0), mid);
        rayon::join(|| matrix_dot_rayon(&a_0, b, &mut c_0),
                    || matrix_dot_rayon(&a_1, b, &mut c_1));

    } else if n > BLOCKSIZE {
        let mid = n / 2;
        let (b_0, b_1) = b.split_at(Axis(1), mid);
        let (mut c_0,
            mut c_1) = c.view_mut().split_at(Axis(1), mid);
        rayon::join(|| matrix_dot_rayon(a, &b_0, &mut c_0),
                    || matrix_dot_rayon(a, &b_1, &mut c_1));
    }
}
```

As before, we first make sure dimensions of `a` and `b` are consistent. Then we enter into the rayon loop. The program recursively divides `a`, `b` and `c` until it reaches the minimum threshold for blocksize, at which point the spawned thread(s) begin multiplication on the blocks of `a` and `b` and update the corresponding blocks of `c` accordingly.

```
test bench_dot_rayon           ... bench:     224,570 ns/iter (+/- 45,483)
test bench_dot_openblas        ... bench:   201,170 ns/iter (+/- 18,666)
```

It now looks like we're competitive with OpenBLAS on my machine.

## Rayon Performance Graph
-------------------

I was interested in how Rayon matrix multiplication performance depended on `BLOCKSIZE` and the overall dimensions of `a` and `b`.

 First,  I looked at two $$ 128 \times 100 $$ and $$ 100 \times 128 $$ matrices. I varied the inner dimension of both matrices in lockstep, and computed benchmarks with/without rayon.

As you can see, with smaller blocksizes, Rayon does consistently worse than single-threaded multiplication.

<img src="http://pegasos1.github.io/public/20160424/fig1.png" width="700">


## Fun experiment on EC2
-------------

I was also interested in how the performance of this matrix multiplication program scaled with compute power. I ran the program on Amazon Web Services EC2 -- a compute optimized c4.2xlarge instance.

A c4.2xlarge  instance has 8 cores of "high frequency Intel Xeon E5-2666 v3 (Haswell) processors optimized specifically for EC2", and 15 GB of RAM.[^9] With high compute power and low extraneous CPU activity, I expected to see far better performance.

Here are the results of multiplying $$ 128 \times 100 $$ and $$ 100 \times 128 $$ matrices on that machine.

```
test bench_dot ... bench: 323,867 ns/iter (+/- 4,005)
test bench_dot_rayon ... bench: 195,585 ns/iter (+/- 15,530)
test bench_dot_openblas ... bench: 91,242 ns/iter (+/- 686)
```

Rayon's performance improved quite a bit! But, it gets blown out of the water by OpenBLAS. No competition here.

We'll dig deeper into these types of tests on various EC2 instances in the last post of this series.


## Things to consider
-----------------------
It's hard to interpret/generalize the results of concurrent programs because, as I showed above, they're highly dependent on a variety of parameters, like the size of the data, the number of threads, the power of the CPU, and the cache size. On top of that, Rayon's *potential parallelism* concept, while really useful for development, makes the overall program a  bit of a black box. For example, why exactly does rayon performance degrade with a smaller `BLOCKSIZE`? Maybe each computation was so ephemeral on a smaller `BLOCKSIZE` that rayon didn't spawn enough threads. Or perhaps I had a random background process that suddenly limited the amount of threads I could allocate. Or perhaps cache lines are the primary determinant of which `BLOCKSIZE` is optimal.

In the next post of this series, we'll go under the hood by investigating the CPU and cache to understand *why* our code improvements resulted in faster performance, hopefully answering some of these questions and controlling for the many factors that can affect the algorithm's speed.

For now, see these results as a showcase of Rayon, as well as an example of how matrix multiplication in Rust can be *really* fast and easy to optimize.

Numeric computing in Rust is exciting. Hope to see more work in this space!


[^1]:[rust-ndarray](https://github.com/bluss/rust-ndarray)
[^2]:[leaf](https://github.com/autumnai/leaf)
[^3]: [ndarray-rblas](https://github.com/bluss/rust-ndarray/tree/master/ndarray-rblas)
[^4]:[matrixmultiply](https://crates.io/crates/matrixmultiply)
[^5]:[matrixmultiply explanation](http://bluss.github.io/rust/2016/03/28/a-gemmed-rabbit-hole/#fn:2)
[^6]: [send and sync](https://doc.rust-lang.org/nomicon/send-and-sync.html)
[^7]: [rayon](http://crates.io/crates/rayon)
[^8]: [rayon explanation](http://smallcultfollowing.com/babysteps/blog/2015/12/18/rayon-data-parallelism-in-rust/)
[^9]: [instance types](https://aws.amazon.com/ec2/instance-types/)
