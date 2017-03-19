---
layout: post
title: Autodifferentiation and Backpropagation
comments: true
---

The derivative is an important operation in machine learning, primarily in the context of numerical optimization on loss functions.
However, since the derivative is an analytic procedure in the continuous realm, its translation to discrete computation has required significant work. This post will detail the various approaches that we can take to compute the derivative of arbitrary functions. First, we'll discuss numerical differentiation, which while simple and intuitive, suffer from floating point rounding errors. Next, we'll discuss symbolic differentiation, which suffers from complexity problems. Last, we'll discuss auto-differentiation, the method of choice to compute derivatives of arbitrary functions with both exactness and simplicity. We'll also discuss *backpropagation*, the method of updating weights in neural networks during learning that is an analogue to auto-differentiation.

## Numerical differentiation

Numerical differentation is rooted in the *finite differences approximation* definition of the derivative. Formally, for a function $$f : \mathbb{R}^N \rightarrow \mathbb{R}$$, we can compute the gradient $$\triangledown f = (\frac{\partial f}{\partial x_1}, \frac{\partial f}{\partial x_2},...,\frac{\partial f}{\partial x_n})$$ as

$$ \frac{\partial f(x)}{\partial x_i} \approx \frac{f(x_i + h) - f(x_i)}{h}$$

where $$h > 0$$ is some small step size.

The issue with numerical differentiation is that it is inherently ill-conditioned and unstable. Essentially, we have to squeeze the infinite representation of real numbers into a finite representation of (usually) 32 or 64 bits. Many real number computations, such as the numerical derivative, cannot be exactly fitted to a finite representation without rounding or truncation, thus introducing approximation errors in the computation.

## Symbolic differentiation

Another popular method of differentiation employed by services like Mathematica is *symbolic differentiation*, the automatic algebraic manipulation of expressions based on well known rules of differentation such as:

$$ \frac{\partial}{\partial x}(f(x) + g(x)) = \frac{\partial f}{\partial x} + \frac{\partial g}{\partial x} $$

or

$$ \frac{\partial}{\partial x}(\frac{f(x)}{g(x)}) = \frac{g(x)\frac{\partial f}{\partial x} - f(x)\frac{\partial g}{\partial x}}{g(x)^2}$$

In symbolic differentiation, expressions are sometimes represented as a trees of symbols that get manipulated by these basic rules to build an exact formula for computation. However, these trees can quickly become complex, leading to a ton of inefficiency.

## Forward Autodifferentiation

Autodifferentiation (AD) is the method of differentiation that is both exact and simple. The basic motivation for AD is that *any arbitrary function can be represented as a composition of simpler ones*. Under functional composition, one can compute the derivative of a function with the *chain rule*:

$$ f(x) = g(h(x)) \implies f'(x) = g'(h(x)) h'(x)$$

Because $$g$$ and $$h$$ are assumed to be simpler components of the arbitrary function $$f$$, their derivatives are also simpler to compute.

An example would help explain this idea.

Let

$$f(x) = \frac{ln(x) + x^2} {sin(x)}$$

Taking the derivative of $$f$$ analytically would be pretty annoying. Fortunately, $$f$$ is a composition of very simple functions:

$$
\begin{align}
& v_0 = ln(x) \\
& v_1 = x^2 \\
& v_2 = v_0 + v_1 \\
& v_3 = sin(x)\\
& v_4 = \frac{v_2}{v_3}\\
& f = v_4
\end{align}
$$

Let $$\dot v_i = \frac{\partial v_i}{\partial x}$$. Differentiation proceeds via a forward pass:

$$
\begin{align}
& \dot v_0 = \frac{1}{x} \\
& \dot v_1 = 2x \\
& \dot v_2 = \dot v_0 + \dot v_1 = \frac{1}{x} + 2x\\
& \dot v_3 = cos(x)\\
& \dot v_4 = \frac{v_3 \dot v_2 - v_2 \dot v_3 }{v_3^2}\\
& \dot f = \dot v_4 = \frac{sin(x)(\frac{1}{x} + 2x) - (ln(x) + x^2)cos(x)}{sin^2(x)}
\end{align}
$$

The important things to notice are that:

  * The forward computation of $$\dot f$$ involves the **staged computation** of very simple components of the function. These stages are trivial to compute.

  * There is a linear flow through the derivative computation, which lends itself well to imperative or iterative programming.  

  * We compute the derivative *exactly*, and do not approximate any value.

Now, let's look at the multi-dimensional case. Let $$f : \mathbb{R}^N \rightarrow \mathbb{R}^M$$.

Then the derivative of $$f$$ is expressed by the Jacobian matrix

$$J_f =

\begin{bmatrix}
\partial f_1 \over \partial x_1 & ... & \partial f_1 \over x_m \\
\vdots  & \ddots & \vdots \\
\partial f_n \over \partial x_1 &  ... & \partial f_n \over \partial x_m \\
\end{bmatrix}
$$

$$J_f$$ can be computed in just $$n$$ forward AD passes across each dimension of $$f$$.

When $$ N << M $$, then forward pass is extremely efficient in computing the Jacobian.

However, when $$ N >> M $$ another version of AD, called *reverse AD*, is more efficent to compute derivatives of $$f$$ with respect to each input dimension. We'll dig into that next.

## Reverse Autodifferentiation

As opposed to forward autodifferentiation, which involves staged computations from the input to the output, reverse autodifferentiation evolves backwards from the function output.

We propagate the derivative of $$f$$ with the *adjoint* operator

$$\bar v_i = \frac{\partial f}{\partial v_i}$$

where $$v_i$$ is some intermediate stage in the overall function computation.  

Reverse AD proceeds in two phases, one of a forward pass computation, and then reverse accumulation. The forward pass, like we outlined in the previous section, helps us keep track of the computational stages that make up the arbitrary function $$f$$. The reverse accumulation then measures the sensitivity of the output of the forward pass with respect to a particular stage (ie, $$\bar v_i$$). We continue to compute derivatives backwards until we arrive at the original input: $$\bar x = \frac{\partial f}{\partial x}$$. Reverse AD is the preferred procedure over Forward AD when the function $$f : \mathbb{R}^N \rightarrow \mathbb{R}^M$$ we're trying to find the derivative of has $$ N >> M $$ because in reverse AD we only have to make $$m$$ passes to compute the multi-dimensional gradient $$f$$.
& \bar v_3 = \bar v_4 \frac{\partial v_4}{\partial v_3} = \bar v_4 \textbf{1} \\
& \bar v_2 = \bar v_4 \frac{\partial v_4}{\partial v_2} = \bar v_4 \textbf{1}\\
& \bar v_1 = \bar v_3 \frac{\partial v_3}{\partial v_1} = \bar v_3 2v_1\\
& \bar v_0 = \bar v_2 \frac{\partial v_2}{\partial v_0} = \bar v_2 \frac{1}{v_0}\\
& \bar x_2 = \bar v_1\\
& \bar x_1 = \bar v_0\\
\end{align}
$$

To compute $$\bar x_1$$ and $$ \bar x_2 $$, we must recognize that $$x_1$$ and $$x_2$$ affect the output $$f$$ in distinct ways.

In fact, it is helpful to view the forward pass as a computation graph, to visualize this point:


$$x_1$$ only affects $$f$$ through $$v_2$$ and $$v_5$$, which means that, through the multi-dimensional chain rule:

$$\frac{\partial f}{\partial x_1} = \frac{\partial f}{\partial v_0} = \frac{\partial f}{\partial v_2}\frac{\partial v_2}{\partial v_0} + \frac{\partial f}{\partial v_5}\frac{\partial v_5 }{\partial v_0} $$

or

$$ \bar v_0 = \bar v_2 \frac{\partial v_2}{\partial v_0} +  \bar v_5 \frac{\partial v_5 }{\partial v_0}$$

On the other hand, $$x_2$$ only affects $$f$$ through $$v_3$$ and $$v_5$$, so:

$$\frac{\partial f}{\partial x_2} = \frac{\partial f}{\partial v_1} = \frac{\partial f}{\partial v_3}\frac{\partial v_3}{\partial v_1} + \frac{\partial f}{\partial v_5}\frac{\partial v_5 }{\partial v_1} $$

or

$$ \bar v_1 = \bar v_3 \frac{\partial v_3}{\partial v_1} +  \bar v_5 \frac{\partial v_5 }{\partial v_1}$$

Thus we show that with reverse AD, we can compute $$\frac{\partial f}{\partial x_1}$$ and $$\frac{\partial f}{\partial x_2}$$ with very elementary operations.

Reverse autodifferentation is known as *backpropagation* in deep learning, and forms the basic way that we update parameters of a neural network during learning. In the next section, we'll dive into how to apply reverse autodifferentation train neural networks.

## Backpropagation in Deep Learning

Consider a two layer neural network. Given an input matrix $$X \in R^{N x M}$$ and output labels $$y \in R^N$$, let $$W_1$$ and $$W_2$$ be weights corresponding to layer 1 and 2 respectively, and $$b_1$$ and $$b_2$$ be bias vectors corresponding to each weight matrix. Between layer 1 and 2 imagine we have a leaky ReLU activation function $$f(x) = max(0, x)$$, and imagine that we apply a softmax classifier after layer 2 to squash its output between $$[0, 1]$$.

The neural network looks like the following:

We can model the neural network as a forward pass of staged computations from the input $$X$$ to the softmax loss function:

$$
\begin{align}
& Y_1 = W_1X + b_1 \\
& A_1 = max(0, Y_1) \\
& Y_2 = W_2A_1 + b_2 \\
& L = SoftmaxLoss(Y_2)\\
\end{align}
$$

How do we define the $$SoftmaxLoss$$ function?

Well, the softmax function applied to a score vector $$x$$ is

$$
p_k = \frac{e^{x_k}}{\sum_j e^{x_j}}
$$

The data loss function for a softmax classifier is

$$ L_i = -log(p_{y_i})$$

The softmax classifier loss for this network is defined as

$$
L = \underbrace{\frac{1}{N}\sum_i L_i}_\text{data loss} + \underbrace{\lambda [\sum_i\sum_j W_1^2 + \sum_i\sum_j W_2^2]}_\text{regularization loss}
$$



During learning, we use gradient descent to optimize the network's weight matrices $$W_1$$ and $$W_2$$. We update the weights with their gradients on $$L$$ , $$\partial L \over \partial W_1$$ and $$\partial L \over \partial W_2 $$.

To find $$\partial L \over \partial W_1$$ and $$\partial L \over \partial W_2 $$, we use backpropagation (or reverse AD).


We've already done the first phase of backpropagation above, the forward pass. Now we compute the backward pass.


$$
\begin{align}
& \bar Y_2 = \bar L \frac {\partial L}{\partial Y_2}\\
& \bar W_2 = \bar Y_2 \frac{\partial Y_2}{\partial W_2} = \bar Y_2 A_1 \\
& \bar b_2 = \bar Y_2 \frac{\partial Y_2}{\partial b_2} = \bar Y_2 \textbf{1} \\
& \bar A_1 = \bar Y_2 \frac{\partial Y_2}{\partial A_1} = \bar Y_2 W_2 \\
& \bar Y_1 = \bar A_1 \frac{\partial A_1}{\partial Y_1} \\
& \bar W_1 = \bar Y_1 \frac{\partial Y_1}{\partial W_1} = \bar Y_1 X \\
& \bar b_1 = \bar Y_1 \frac{\partial Y_1}{\partial b_1} = \bar Y_1 \textbf{1}
\end{align}
$$

So our task is to find $$\bar W_1$$ and $$\bar W_2$$.

Let's start with $$\bar W_2$$.

$$\bar W_2 = \bar Y_2 A_1 = \bar L \frac {\partial L}{\partial Y_2} A_1 = \frac {\partial L}{\partial Y_2} A_1$$

To begin the backpropagation, we first have to find $$\frac{\partial L}{\partial Y_2}$$. $$Y_2$$ is just an unnormalized score vector. We will set an alias $$f  = Y_2$$. Consider first the data loss, $$ H_j = \frac{1}{N}\sum_i L_i = -\frac{1}{N}\sum_i log(p_j)$$.

When $$ j = y_i $$,

$$ \frac{\partial H}{\partial f_{y_i}} = -\frac{1}{N} \sum_i log(p_{y_i}) \frac{\partial p_{y_i}}{\partial f_{y_i}}$$

$$  = -\frac{1}{N} \sum_i \frac{1}{p_{y_i}}  \frac{\sum_j e^{f_{j}} e^{f_{y_i}} - e^{f_{y_i}} e^{f_{y_i}} } {\sum_j e^{f_j}}$$

$$  = -\frac{1}{N} \sum_i \frac{1}{p_{y_i}} \frac{e^{f_{y_i}} (\sum_j e^{f_{j}}  - e^{f_{y_i}})}{\sum_j e^{f_j}}$$

$$  = -\frac{1}{N} \sum_i \frac{1}{p_{y_i}} p_{y_i}(1 - p_{y_i})$$

$$  = p_{y_i} - 1$$



When $$ j \neq y_i $$,

$$ \frac{\partial H}{\partial f_{j}} = -\frac{1}{N} \sum_i log(p_{y_i}) \frac{\partial p_{y_i}}{\partial f_{j}}$$

$$  = -\frac{1}{N} \sum_i \frac{1}{p_{y_i}} \frac{0 - e^{f_{y_i}}e^{f_{j}}}{\sum_j e^{f_j}}$$

$$  = -\frac{1}{N} \sum_i \frac{1}{p_{y_i}} (- p_{y_i}p_j)$$

$$  = p_j$$

So this means:

$$
\triangledown H_j =
\begin{cases}
  p_j & \text{if } j\neq y_i \\    p_{j} - 1  & \text{if } j=y_i\
\end{cases}
$$

In other words,

$$
\triangledown H_k =  p_k - \mathbb{1}(k = y_i)
$$

Where 1 is just a binary indicator function that evaluates to 1 when the predicate is true.

Taking into account the gradient of the regularization term in $$L$$,

$$
\triangledown L = \frac{\triangledown H}{N} + \lambda\sum\limits_{i}\sum\limits_{j}W_{ij}
$$

Basically to get the derivative of $$L$$, we just subtract its output with a binary indicator that is set to 1 when the index is the one of the true class.


## Code 'em up

As mentioned before, autodifferentiation lends itself very well to iterative programming. Here we'll create a neural network in Python and update its parameters during learning with backpropagation.


First, we perform the forward pass, computing the class scores for the input.

```python
import numpy  as np

Y1 = X.dot(W1) + b1 # first layer
A1 = np.maximum(0, Y1) # leaky ReLU
Y2 = A1.dot(W2)+ b2 # second layer
exp_Y2 = np.exp(Y2)
scores = exp_Y2 / np.sum(exp_Y2, axis=1, keepdims=True) # softmax
```

Then, we compute the loss:

```python
import numpy  as np

data_loss = -np.log(prob[range(N), y])
reg_loss = 0.5 * reg * np.sum(W1 * W1)  + 0.5 * reg * np.sum(W2 * W2)
loss = np.sum(data_loss) / N + reg_loss
```

Next we compute the backpropagation, computing the derivatives of the weights and biases.

```python
import numpy  as np

grads = {}

dscores = scores
dscores[range(N),y] -= 1
dscores /= N

grads['W2'] = a1.T.dot(dscores)
grads['b2'] = np.sum(dscores, axis = 0)

dhidden = dscores.dot(W2.T)
dhidden[a1 <= 0] = 0

grads['W1'] = X.T.dot(dhidden)
grads['b1'] = np.sum(dhidden, axis = 0)

grads['W2'] += reg * W2
grads['W1'] += reg * W1
```
