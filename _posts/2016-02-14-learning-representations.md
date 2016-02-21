---
layout: post
title: Learning Representations
---

*This post is 1st in a series on representation learning.*

I've stated before that feature engineering is often the most time-consuming
and difficult process of building machine learning services.[^1] The reality is that many learning algorithms are dependent on the supply of diverse and balanced training data to perform well, and this type of data is unavailable in most contexts. In the case of security, even building a simple classifier of good and bad URLs requires an incredible effort to gather malicious and benign sources of URLs, which are, suprisingly, hard to come by. 

Furthermore, feature engineering is mediated by humans, and we're biased and limited in scope.

So what makes a representation *good*? In the following, let $$f$$ be some learning algorithm.

### Smoothness and predictability

$$ || f(x) - f(y) || \le || x - y || $$


### Natural clusters

### Invariance

$$f(x) \approx f(y) \implies f(x + \epsilon) \approx f(y) $$

In other words, the representation must be able to withstand small perturbations of the data.

### Multiple, balanced explanatory factors

$$ f(x) = F x = f_1 x + f_2 x + f_3 x + ...  \text{ where } n << \infty $$

We wan
### Feature disentanglement

$$ det(F) = 0 $$


### Expressive power

Not only should features be independent from one another, but they should also span the basis.

These criteria are very difficult to come by for most problems.

What if we could automate feature engineering? This is the basic premise around *representation learning*, a sub-discipline of machine learning that involve algorithms that find the best representation of data to feed into a model.

This series of blog posts will revolve around algorithms that help us find the best representation of data to feed into a model.


[^1]: [blog post at rapid7 community](https://community.rapid7.com/community/infosec/blog/2016/01/04/applying-machine-learning-to-security-problems)
