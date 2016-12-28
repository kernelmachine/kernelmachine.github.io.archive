---
layout: post
title: Introducing Utah
comments: true
---


Recently, I've been working on a Rust crate for tabular data manipulation. I'm excited to share the current state of the project.

## A dataframe for Rust

Languages like [Python](http://pandas.pydata.org/), [R](http://www.r-tutor.com/r-introduction/data-frame), and [Julia](https://github.com/JuliaStats/DataFrames.jl) have popularized the  _dataframe_, which is an abstraction over a collection of _named_ arrays. The dataframe allows users to access, transform, and compute over two-dimensional data that may or may not have mixed types.

[**Utah**](https://github.com/pegasos1/utah) is a dataframe crate backed by [ndarray](https://github.com/bluss/rust-ndarray) for type-conscious tabular data manipulation with an expressive, functional interface.

## Overall goals of project

#### Functional interface to transform data

Transformations are composable, repeatable, and readable. This can be a real boon in preventing pipeline jungles that are hard to parse.

#### Iterator adapter implementations for performance and laziness

Data transformations are implemented with Rust's iterators, which are fast, safe, and lazy.

#### Leverage reference locality for performance

Most implementations of the dataframe involve some sort of map between column names and the underlying data. You're likely incurring performance hits while chasing pointers around in memory. In particular, row-wise operations can be much slower than column-wise ones.

I wanted to explore the effects of keeping your data close together in memory, while supporting mixed types. This crate is backed by ndarray to hold data, and enums to support mixed types. At the end of this post, I'll describe an alternate implementation that sacrifices data locality to be able to have mixed types without wrappers.

#### Solid error handling

This an area of the project that I haven't completed yet, but has a ton of potential. There are many errors that can occur while querying dataframes. For example, imagine selecting a column that doesn't exist in the dataframe, or joining two dataframes that don't have a common key. Such simple errors can be hidden in a sea of complex data transformations, potentially leading to a lot of technical debt when building data pipelines. The goal here is to provide compile-time error handling of these sorts of mistakes. I explore potential avenues to accomplish this goal at the end of this post.

## Internals

*Note: The internals of the project are subject to change in the future; I'll try to keep an updated post describing changes.*

### The DataFrame

There are two core types in Utah: the **DataFrame** and the **DataFrameIterator**.

Utah dataframes are defined as follows:

```rust
pub struct DataFrame<T>
    where T: UtahNum,
          // ^ math ops like Add/Mul/etc, Empty, and others.
{
    pub columns: Vec<String>,
    pub data: Matrix<T>,
    pub index: Vec<String>,
}
```

The `DataFrame` takes the generic parameter *T*, which corresponds to the type of inner matrix of data.

The `DataFrame` is read-only by default. To operate on a dataframe that is read-write, you can use a `DataFrameMut`:

```
pub struct DataFrameMut<'a, T>
    where T: 'a + UtahNum
{
    pub columns: Vec<String>,
    pub data: MatrixMut<'a, T>,
    pub index: Vec<String>,
}
```

The only thing we've changed is the `data` field -- from a `Matrix<T>` to a `MatrixMut<T>`. For simplicity, we'll disregard mutable dataframe for the rest of this post, but know that everything we talk about below extends to the mutable variant.


The inner data is of type `Matrix<T>`, which is a type alias to a [2-d array](http://bluss.github.io/rust-ndarray/master/ndarray/type.Array2.html), from the ndarray crate.

```rust
pub type Matrix<T> = Array2<T>;
```


The data that the Matrix contains implement all the traits associated with the custom trait *UtahNum* (ie `Add`,`Sub`,`Mul`,`Div`, `One` and `Zero`) for computations, as well as the custom trait *Empty*.

Empty values are an unfortunate reality of most datasets. We use the `Empty` trait to define empty values for any type we want to use in a dataframe:

```rust
pub trait Empty<T> {
    fn empty() -> T;
    fn is_empty(&self) -> bool;
}
```

We ask, for type `T`, what should we consider as an empty value, and when is a value equal to empty? In the case of `f64`, empty values will usually be `NaN`, so we might implement it as follows:

```rust
impl Empty<f64> for f64 {
    fn empty() -> f64 {
        NAN
    }
    fn is_empty(&self) -> bool {
        self.is_nan()
    }
}
```

The columns/index are just names of the columns and rows of the dataframe, respectively.

### The DataFrameIterator

The `DataFrameIterator` is what we use to perform dataframe transformations and computations. The `DataFrameIterator` is of the following type:

```
pub struct DataFrameIterator<'a, I, T: 'a>
    where I: Iterator<Item = Window<'a, T>>
{
    pub names: Iter<'a, String>,
    pub data: I,
    pub other: Vec<String>,
    pub axis: UtahAxis,
}
```


First up, notice that we've now got lifetimes in the type. The DataFrameIterator is just a reference to a dataframe that owns the original data, and cannot outlive it. Ideally, you'll have to read the data into memory only once.

Next, the trait bound on the dataframe iterator is `Iterator<Item = Window<'a, T>>`. `Window<'a, T>` is another type alias:

```
pub type Window<'a, T> = (String, ArrayView1<'a, T>);
```

The `ArrayView1` [type](`http://bluss.github.io/rust-ndarray/master/ndarray/type.ArrayView1.html`) is a column or row-wise slice of the original data. The `Iterator` bound tells us that the `DataframeIterator` iterates over _views_ of the data, along with their names (i.e. a column or index value).  The dataframe lives in a contiguous area of memory, and to iterate over it, we just slide a window over the stride of the vector that represents the matrix containing the data. The iterator is realized through ndarray's `AxisIter`[type]("http://bluss.github.io/rust-ndarray/master/ndarray/struct.AxisIter.html").

Finally, the struct fields:

* `data` just houses the iterator over the "windows" we just discussed.

* `names` is an abstraction over the dataframe's rows or columns, depending on how we're iterating over the data during transformation. If we're iterating over the dataframe column-wise, `names` will be an iterator over the `columns` field. If we're iterating row-wise, `names` will be an iterator over the `index` field.

* `other` houses the axis label that you're *not* iterating over. For example, in the case of a column-wise dataframe iterator, `other` would be the original index. We just hold onto this value in case you want to allocate the transformation into a new dataframe.

* `axis`, which is of type `UtahAxis`, is an enum over two different directions of iteration:

```rust
pub enum UtahAxis {
    Row,
    Column,
}
```

This tells the dataframe iterator which direction you want to iterate over. For example, if you eventually want to take a sum of the columns, you'd use the `UtahAxis::Column` when creating the dataframe iterator.

Now, ideally dataframes should be able to support mixed types. I'll discuss how Utah handles this below. But an important thing to note is that because strings are not `Copy`, the dataframe is not `Copy`. This means that you may run into unexpected ownership problems while building data transformations. There are some API ergonomics to be ironed out.


## Creating a dataframe

There are multiple ways to create a dataframe. The most straightforward way is to use a *builder pattern*, which allows you to overwrite individual fields of the Dataframe successively:

```

use utah::prelude::*;
let c = arr2(&[[2., 6.], [3., 4.], [2., 1.]]);
let df: DataFrame<f64> = DataFrame::new(c)
                                  .columns(&["a", "b"])?
                                  .index(&["1", "2", "3"])?;
```

The `?` operator, newly introduced syntax for accessing the `Result` type, is there to prevent you from adding columns or indices that don't match the dimensions of the underlying data.

There's also `dataframe!` and `col!` macros which you can use to create new dataframes on the fly:

```
let k: DataFrame<f64> = dataframe!(
    {
        "a" =>  col!([2., 3., 2.]),
        "b" =>  col!([2., NAN, 2.])
    });
```

Finally, you can import data from a CSV.

```
let file_name = "test.csv";                            
let df: Result<DataFrame<InnerType>> = DataFrame::read_csv(file_name);
```

Note that Utah's `ReadCSV` trait is pretty barebones right now.


### Combinators

The user interacts with Utah dataframes by chaining combinators, which are essentially iterator extensions (or _adapters_) over the original dataframe. This means that each operation is lazy by default. You can chain as many combinators as you want, but it won't do anything until you invoke a collection operation like `as_df`, which would allocate the results into a new dataframe, or `as_matrix`, which would allocate the results into an ndarray matrix.

I've organized the combinators that I've built so far into four different types, but there are naturally many more. The nice thing is that the iterator adapter design makes it extremely easy to add new combinators to the project.

#### Transform combinators

These combinators are meant for changing the shape of the data you're working with. Combinators in this class include `select`, `remove`, and `append`.

```
let a = arr2(&[[2.0, 7.0], [3.0, 4.0], [2.0, 8.0]]);
let df = DataFrame::new(a).index(&["a", "b", "c"]).unwrap();
let res = df.select(&["a", "c"], UtahAxis::Row);
```


#### Process combinators

Process combinators are meant for changing the original data you're working with. Combinators in this class include `impute` and `mapdf`. Impute replaces missing values of a dataframe with the mean of the corresponding column. Note that these operations require the use of a `DataFrameMut`.

```
let mut a: DataFrameMut<f64> = dataframe!(
    {
        "a" =>  col!([NAN, 3., 2.]),
        "b" =>  col!([2., NAN, 2.])
    });
let res = df.impute(ImputeStrategy::Mean, UtahAxis::Column);
```


#### Interact combinators

Interact combinators are meant for interactions between dataframes. They generally take at least two dataframe arguments. Combinators in this class include `inner_left_join`, `outer_left_join`, `inner_right_join`, `outer_right_join`, and `concat`.

```
let a: DataFrame<f64> = dataframe!(
    {
        "a" =>  column!([NAN, 3., 2.]),
        "b" =>  column!([2., NAN, 2.])
    });
let b: DataFrame<f64> = dataframe!(
    {
        "b" =>  column!([NAN, 3., 2.]),
        "c" =>  column!([2., NAN, 2.])
    });
let res = a.inner_left_join(&b).as_df()?;
```

#### Aggregate combinators

Aggregate combinators are meant for the reduction of a chain of combinators to some result. These combinators are usually the last operation in a chain, but don't necessarily have to be. Combinators in this class include `sumdf`, `mindf`, `maxdf`, `stdev` (standard deviation), and `mean`. Currently, aggregate combinators are not iterator collection operations, because they do not invoke an iterator chain. This may change in the future.

```
let a = arr2(&[[2.0, 7.0], [3.0, 4.0], [2.0, 8.0]]);
let df = DataFrame::new(a).index(&[1, 2, 3])?.columns(&["a", "b"])?;
let res = df.mean(UtahAxis::Row);
```

#### Chaining combinators

The real power in combinators come from the ability to chain them together in expressive transformations that are easy to parse. You can do things like this:

```
let result = df.df_iter(UtahAxis::Row)
               .remove(&["1"])
               .select(&["2"])
               .append("8", new_data.view())
               .inner_left_join(df_1)
               .sumdf()
               .as_df()?
```

Because we've built the chain on a row-wise dataframe iterator, each subsequent operation will only operate on the rows of the dataframe.

## Creating new combinators

Rust's trait system allows for repeatable patterns throughout the project, especially when it comes to designing combinators. By following these steps, you can easily add your own combinators for new transformations or computations.

##### 1. Define a new struct `Combinator`, with necessary data.

The combinator should contain an iterator over items of type `Window<'a, T>` -- a matrix slice and its name.  `Sum` contains the Iterator, the axis label it's not iterating over, and the axis of iteration.

```
    pub struct Sum<'a, I: 'a, T: 'a, S>
      where I: Iterator<Item = Window<'a, T>> + 'a,
            T: UtahNum,
            S: Identifier
  {
      data: I,
      other: Vec<String>,
      axis: UtahAxis,
  }
```

##### 2. impl Iterator for Combinator

What's the output of this combinator during iteration over the dataframe? In the case of `Sum`, we're just taking the scalar sum of elements in a row or column, depending on the axis of iteration.


```
      impl<'a, I, T> Iterator for Sum<'a, I, T>
        where I: Iterator<Item = Window<'a, T>>,
              T: UtahNum    
    {
        type Item = T;
        fn next(&mut self) -> Option<Self::Item> {
            match self.data.next() {
                None => return None,
                Some((_, dat)) => return Some(dat.scalar_sum()),
            }
        }
    }
```

##### 3. Add `impl Combinator` with a `fn new()`.

How do you create this combinator from scratch? That's easy:

```
      impl<'a, I, T> Sum<'a, I, T>
        where I: Iterator<Item = Window<'a, T>>,
              T: UtahNum
    {
        pub fn new(df: I, other: Vec<String>, axis: UtahAxis) -> Sum<'a, I, T> {
            Sum {
                data: df,
                other: other,
                axis: axis,
            }
        }
    }
```

##### 4. Add `fn combinator(self : Iterator)` to its categorical trait

The function you add should invoke the `new` constructor defined in the third step. In the case of `Sum`, we'll add it to the `Aggregate` trait. Define a new categorical trait if the new combinator doesn't fit with the built-in ones.

```
pub trait Aggregate<'a, T>
    where T: UtahNum
  {
    fn sumdf(self) -> Sum<'a, Self, T>
        where Self: Sized + Iterator<Item = Window<'a, T>>;
  }
```


##### 5. Add `fn combinator(df : DataFrame)` to the `Operations` trait

The function you add should invoke the combinator from an allocated dataframe.

```
  pub trait Operations<'a, T>
    where T: 'a + UtahNum
  {
    fn sumdf(&'a mut self, axis: UtahAxis) -> SumIter<'a, T>;
  }
```

##### 6. impl AsDataFrame for Combinator

The function you add will let you go from the `Combinator` iterator to an allocated dataframe/matrix/array. Read next section for more details on this trait.

```
  pub trait ToDataFrame<'a, I, T>
    where T: UtahNum + 'a
  {
          fn as_df(self) -> Result<DataFrame<T>> where Self: Sized + Iterator<Item = I>;
  }
```

It's interesting to think about how to reduce combinators to iterators adapters that we're familiar with. For example, the `concat` combinator is just a `Chain`. More complicated combinators like `Groupby` can be thought of as a `Map` of a `Filter`. These conceptual relationships can help you think about how to best implement a new combinator.

### Collection

There are many ways you can access or store the result of your chained operations. Because each data transformation is just an iterator, we can naturally collect the output of chained operations via `collect()` or a `for` loop:

```
for x in df.concat(&df_1) {
   println!("{:?}", x)
}
```

But we also have an `AsDataFrame` trait, which dumps the output of chained combinators into a new dataframe, matrix, or array, so we can do something like the following:


```
let maximum_values = df.concat(&df_1).maxdf(UtahAxis::Column).as_df()?;
```


### The InnerType

Now, I mentioned in the beginning that most dataframes provide mixed types, and I wanted to provide a similar functionality here. In the module `utah::mixedtypes`, I've defined `InnerType`, which is an enum over various types of data that can co-exist in the same dataframe:

```
pub enum InnerType {
    Float(f64),
    Int64(i64),
    Int32(i32),
    Str(String),
    Empty,
}
```

With this wrapper, you can have Strings and f64s in the same dataframe.

In general, this may not be the best approach to supporting mixed types in this project, because are performance hits for computation when working with type wrappers. I discuss alternative avenues to achieving this goal below.


## Next steps

#### Comparing a map implementation to a reference-local one

There are drawbacks to the reference-local implementation of the dataframe, mainly that we have to wrap our data with an enum (ie `InnerType`) to be able to support mixed types. Computations are not as fast as they could be.


Ideally, we could work with raw types directly. An alternate dataframe implementation sacrifices reference locality for the ability to work with raw types in the mixed-context.

```rust
pub struct MixedDataFrame<T>
    where T: UtahNum
{
    pub data: BTreeMap<String, Row<T>>,
    pub index: BTreeMap<String, usize>,
}
```

Here we use a `BTreeMap` to maintain order across columns. `Row<T>` is a type alias to the 1-D ndarray:

```rust
pub type Row<T> = Array1<T>;
```

This dataframes design implies that we will need two separate types for row-wise and column-wise iteration.

Row-wise iteration would manifest in terms of a multi-zip operation where we connect each value of the same index across all columns.

```
pub struct MixedDataFrameRowIterator<'a, T: 'a>
    where T: UtahNum
          Zip<AxisIter<'a, T, usize>>: Iterator
{
    pub names: Iter<'a, String>,
    pub data: Zip<AxisIter<'a, T, usize>>,
    pub other: Vec<String>,
    pub axis: UtahAxis,
}
```

On the other hand, column-wise iteration would just be an iterator over the BTreeMap.

```
pub struct MixedDataFrameColIterator<'a, T: 'a>
    where T: UtahNum,
          S: Identifier,
          BTreeIter<'a, String, Row<T>>: Iterator
{
    pub data: BTreeIter<'a, String, Row<T>>,
    pub other: Vec<String>,
    pub axis: UtahAxis,
}
```

`BTreeIter` is a type alias to the BTreeMap iterator.

#### Better Error handling with compile time dimension checking

How do we effectively prevent user errors during data transformation _at compile time_?

So far, the only error handling is in the `new()` function, where we check whether the dimensions of the axis labels match that of the underlying data. This actually handles a ton of common errors, like selecting or removing a column that doesn't exist. But while it does panic in the right situations, the errors are not that useful in figuring out what the actual mistake was:

```
IndexShapeMismatch(expected: String , actual: String) {
    display("index shape mismatch. Expected length: {}, Actual length: {}",  expected, actual)
}

ColumnShapeMismatch(expected: String , actual: String) {
    display("column shape mismatch. Expected length: {}, Actual length: {}",  expected, actual)
}

```

Furthermore, the errors are only caught at runtime.

One thing I've been thinking about is compile-time dimension checking with the [typenum](https://github.com/paholg/typenum) and [genericarray](https://github.com/fizyk20/generic-array) crates, which provide compile-time numeric operations and array dimensions, respectively.

Using these crates, the dimensions of the inner data and axis labels could be embedded in trait bounds:

```
pub struct DataFrame<T, N, M, O, P>
    where T: UtahNum
          M: ArrayLength<String> + Same<P>,
          P: ArrayLength<T> + Same<M>,
          N: ArrayLength<String> + Same<O>,
          O: ArrayLength<T> + Same<N>,
{
    pub columns: GenericArray<String, M>,
    pub data: GenericArray<GenericArray<T,O>, P>,
    pub index: GenericArray<String, N>,
    phantom_0: PhantomData<<N as Same<O>>::Output>,
    phantom_1: PhantomData<<P as Same<M>>::Output>,
}
```

The trait bounds imply that a combinator chain will not compile if the length of the index `M` and columns `N` don't match the dimensions of the underlying data.

#### Streaming DataFrames


In reality, the current dataframe implementation is an imperfect workaround of what I _really_ want the dataframe and its combinators to be. The data owner (`DataFrame`) is separated from the iterator (`DataFrameIterator`) and the combinators. This is because it's impossible to return values whose lifetime is tied to the iterator itself. I would like the dataframe to be an iterator over some data held in disk, and each combinator borrowed values from a buffer maintained by the dataframe.  Then the real power of the iterator adapter design is realized: we can work with datasets that may not fit into memory.

What I'm essentially talking about is _streaming iterators_, which has been discussed at length [here](https://users.rust-lang.org/t/returning-borrowed-values-from-an-iterator/1096) and [here](https://github.com/emk/rust-streaming). There's another [interesting crate](https://github.com/sfackler/streaming-iterator) around this effort too. It's an exciting concept.

## Conclusion

I've introduced a new Rust crate for handling complex data transformations with two dimensional iterator adapters. This crate, while extremely nascent, has shown a few strengths so far:

  1. Functional chaining of combinators makes code easy to understand.
  2. Iterator adapter design makes the code extremely repeatable and easy to extend to new domains.
  3. We get all the benefits of working with Rust's iterators, which are fast, safe, and lazy.
  4. There's a lot of interesting future work on better mixed type support, compile time numbers, and streaming.

That's all from me; happy holidays!
