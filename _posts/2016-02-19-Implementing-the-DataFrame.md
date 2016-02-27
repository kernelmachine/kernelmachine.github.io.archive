---
layout: post
title: Implementing the DataFrame in Rust
comments: true
---

The pandas dataframe is a horrible data structure from a type perspective. It's a structure
in which floats, ints, and strings can co-exist. This makes the concept of a dataframe less straightforward
to port to type-safe languages like Rust. 
