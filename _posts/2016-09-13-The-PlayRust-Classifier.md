---
layout: post
title:  The PlayRust Classifier
comments: true
---


*For those who weren't able to attend [RustConf 2016](http://rustconf.com/), I thought I'd provide a synopsis of my and /u/staticassert's talk. I've only focused on a subset of talking points below; for the full talk, check out the upcoming video when it's posted.*

## Technical Debt in Data Science

Our RustConf talk, entitled *[The PlayRust Classifier](https://slides.com/suchingururangan/playrustclassifier)*, was essentially about how to reduce technical debt. It's something that most software engineers experience regularly -- the lack of documentation, unhandled errors, costly scaling, etc. However, technical debt in  machine learning compounds quickly due to unique challenges in the space.

Most data science teams face the basic problem of moving between research and production-level machine learning. These two types of data science have very different motivations and goals. Research data science is usually one-off and includes a lot of proof-of-concept work, while production-level machine learning may run in the cloud or on a low-memory device, and has the same engineering constraints as any other software product. Most technical debt is accrued in the transition between these phases, especially during feature engineering.

<div style="margin-left:175px">
  <img src="http://pegasos1.github.io/public/20160913/datapic.jpg" width="400" height="400">
</div>

The unfortunate reality is that data often sucks in most applicable domains of machine learning.   This means that data scientists spent 99% of their time doing feature engineering -- munging through data, building and cleaning features -- and only a minority of their time working with machine learning algorithms.

Tech debt during feature engineering comes in many flavors, as outlined by *[Machine Learning: The High Interest Credit Card of Technical Debt (2014)](http://research.google.com/pubs/pub43146.html)*, a crucial motivator of our talk:

1) **Siloed Teams**: Data scientists and software engineers are usually considered distinct teams. This means that POC research code may not be held up to common engineering standards, and that handoff to production may involve a lot of reimplementation. Furthermore, siloed teams can make ML models more esoteric to those who are tasked with making them production-ready.

2) **Pipeline Jungles**: Complex transformations of data to get it into an ML-friendly format means that the related code can be extremely hard to reason about, and thus difficult to move into production. Extraneous, messy supporting code can leak into your codebase, especially in dynamic languages that give you more freedom over the way you represent and manipulate data. Pipeline jungles can prevent errors from bubbling up, making it very difficult to trace where things are breaking.

3) **Unscalable Experiments**: Code handed off to engineers may be not only monolithic and hard to reason about, but also difficult to scale. Writing code that is difficult to parallelize may lead to costly horizontal scaling in the cloud.

The long story short is that machine learning services are not only dependent on the quality of machine learning models, but also the quality of feature engineering and data ingestion. This realization prompted us to look for tools that would help us become more confident that feature engineering pipelines are reliable and robust in a very non-deterministic domain.

## The /r/playrust classifier

How can Rust help us pay off technical debt in machine learning during the feature engineering stage? To explore the strengths of Rust in this area, my co-speaker and I built a classifier to solve a well-known problem for the Rust reddit community.

From time to time, someone mistakenly publishes a post in the *[/r/rust](http://reddit.com/r/rust)* subreddit that was intended for the *[/r/playrust](http://reddit.com/r/playrust)* subreddit, a community for a popular video game *Rust*. We built a classifier to detect these mistakenly published posts.

<div style="text-align:center">
  <img src="http://pegasos1.github.io/public/20160913/playrust.png" width="780" height="400">
</div>

This toy problem was an optimal medium to explore Rust data science, because we were gifted with naturally labeled training data: posts collected from both subreddits. This let us focus on the implementation details.


### The Model

Before digging into Rust-specific features of the pipeline we built, let's quickly look at the resulting model and its accuracy.

We gathered thousands of reddit posts and looked at a number of features to describe their respective subreddits:

* Author popularity
* Upvotes
* Downvotes
* Post length
* Word frequency
* Symbol frequency
* Regex matches on Rust code

We then trained a Random Forest with the crate *rustlearn* to perform the predictions.


### Results

We achieved good accuracy in our model. The model had a >98% AUC in prediction, as seen below.

<div style="margin-left:100px">
  <img src="http://pegasos1.github.io/public/20160913/roccurve.png" width="550" height="400">
</div>

The model was primarily driven by the frequencies of words related to the */r/rust* subreddit. Notice the word "amp", ie "&". Haha.

<div style="margin-left:100px">
  <img src="http://pegasos1.github.io/public/20160913/featimps.png" width="580" height="400">
</div>

Some example outputs are below. Notice the third post, with the slightly confusing title, is also slightly confusing to the model.

<div style="margin-left:50px">
  <img src="http://pegasos1.github.io/public/20160913/examples.png" width="700" height="400">
</div>


## Rust advantages

So what was our experience using Rust to build this model end-to-end? Did Rust showcase strengths in reducing technical debt?

### Upfront error handling

A powerful aspect of the Rust language is the idea that developers must handle potential errors upfront.

```rust
pub fn get_reddit_post(&self, url : &str) -> Vec<RawPostData> {
   let mut res = self.client
                     .get(url)
                     .send()
                     .unwrap();

   let data = extract_data(&mut res)
                  .unwrap();
   data
}
```

In the above code, we send a `GET` request to a url, and extract data from it. You can see that you *must* handle the potential errors that could surface from each of these operations. For example, the network could go down, or data extraction may fail for some reason. We easily handle potential errors with an *unwrap()* method, in which we assert that we are sure that this method won't fail on us. This may be something you see in POC research code. We don't really need anything fancy here.

```rust
pub fn get_reddit_post(&self, url : &str) -> Result<Vec<RawPostData>> {
     let mut res = try!(self.client
                            .get(url)
                            .send()
                            .chain_err(||
                                format!("Failed to GET {}", url)));


     let data = try!(extract_data(&mut res)
                    .chain_err(||
                          format!("Failed to parse data {}", url)));
     Ok(data)
}
```

But in production, we definitely want to handle any potential errors in some meaningful way. We can do this with the `try!` macro.

This approach to handling errors is different from the unchecked exceptions paradigm in languages like Python or Java. Unlike try/except, `try!` is precise. In the former paradigm, you tend to wrap larger codeblocks with try/except, when in reality only parts of the code may fail.

Furthermore, potential errors are  baked into the output type of the function (the `Result` type). This means that proper error handling can occur *without knowledge of function implementation*. On the developer's side, it's easy to move from research code to production, just `Ctrl+F` the `unwraps`, and handle them with `try!` macros.

### Typed approach to Dataframes

Dataframes are a tabular data format for many languages like Python, R, and Julia. Dataframes are ergonomic, but can lead to technical debt, by allowing things like mixed types per column. This could lead to bugs in which unexpected values crop up where they're not supposed to -- a big headache to trace in large datasets.

In static languages like Rust, we do care about types associated with our data. For the */r/playrust* classifier, we stored raw data collected about the subreddits in a struct called `RawPostData`:

```rust
struct RawPostData {
    is_self: bool,
    author_name: String,
    url: String,
    downvotes: u64,
    upvotes: u64,
    score: u64,
    edited: bool,
    selftext: String,
    subreddit: String,
    title: String,
}
```

And extracted features were stored in a struct called `ProcessedPostFeatures`:

```rust
struct ProcessedPostFeatures {
   is_self: f32,
   author_popularity: f32,
   downs: f32,
   ups: f32,
   score: f32,
   post_len: f32,
   word_freq: Vec<f32>,
   symbol_freq: Vec<f32>,
   regex_matches: Vec<f32>,
}
```

```rust
fn main() {
    let v : Vec<RawPostData> = get_raw_data();

    v.iter()
        .map(|post| post.author)
        .map(|author| calculate_author_value(author))
        .collect()
}
```

Each field of the struct was equivalent to a dataframe column, and each index of the field was equivalent to a dataframe row. This typed approach gave us confidence that we did not populate unexpected values in our dataframe for a particular column. We could apply transformations in our data in the normal way we map over iterators.

### Parallelization

Furthermore, this structure to dataframes allowed us to easily parallelize operations with crates like *rayon*. We just change `iter` to `par_iter` in the code above, and we're golden.

```rust
extern crate rayon;

use rayon::prelude::*;

fn main() {
    let v: Vec<RawPostFeatures> = get_raw_features();
    let processed: Vec<f64> = Vec::with_capacity(v.len());

    v.par_iter()
        .map(|post| post.author)
        .map(|author| calculate_author_value(author))
        .collect_into(&mut processed)
}
```

### Other advantages

There are many other very useful aspects of Rust for the data science pipeline, including:

* Predictable performance during scaling
* Cargo testing, benchmarking, and docs help devs follow good coding practices
* Trait composition/generics limit the need for messy glue code
* Many benchmarks (like [this](http://www.suchin.co/2016/04/25/Matrix-Multiplication-In-Rust-Pt-1/) one) suggest Rust's strong performance in numerics

## Rust disadvantages

While using Rust was generally a pleasant experience for this project, there were some areas in which the language fell short.

### Fragmented ML ecosystem

The reality is that the current machine learning community is sparse. We found that while there are 60+ crates on crates.io associated with machine learning or linear algebra, many of these libraries provide similar functionality with different APIs. For example, most machine learning tools have custom matrix implementations. This limits the interoperability of crates that makes a language like Python, which builds most of its numeric libraries around the numpy array, very attractive for data science.

### Data exploration difficult in a static language

Dynamic languages like Python and R dominate data investigation, and with good reason. A REPL/interpreter lends itself very well to exploration, because you have instant feedback to tweaks in your code -- you don't need to re-compile to see effects. Furthermore, during the data investigation stage, performance is not that important, so we can get away with ignoring language level details that may slow us down while exploring. Last, Python and R are laden with libraries around graphing and visualization, which is totally non-existent in Rust. Most mature machine learning systems are hybrids of languages and tools specialized for specific tasks, and we envision Python and R still dominating this space.

## Vision for Rust ML

We have shown that Rust language features help reduce many technical debt issues that arise in making production level data science systems. We hope that Rust is promoted to improve feature engineering systems. We also hope that implementations of data science tooling become standardized to facilitate interoperability. Finally, we believe that effective domain applications of a language are primarily driven by the community that forms around it. We should start sharing ideas and building a collective metric for success in Rust machine learning and numerics.
