---
layout: post
title: Automatic differentiation and Backpropagation
comments: true
---

*Much of this blog post was inspired by [CS231n](http://cs231n.github.io) and [this](https://arxiv.org/abs/1502.05767) paper. Highly recommended reads.*

The derivative is an important operation in machine learning, primarily for parameter optimization with loss functions. However, computing analytic derivatives in the finite context of 32 or 64 bits has required significant work. This post will detail the various approaches that we can take to compute the derivative of arbitrary functions. First, we'll discuss numerical differentiation, which, while simple and intuitive, suffers from floating point errors. Then, we'll discuss symbolic differentiation, which suffers from complexity problems. Finally, we'll discuss auto-differentiation, the most popular method to compute derivatives with both exactness and simplicity. We'll also discuss *backpropagation*, an analogue of auto-differentiation that is the primary method of learning in neural networks.

## Numerical differentiation

Numerical differentation is rooted in the *finite differences approximation* definition of the derivative. Formally, for a function $$f : \mathbb{R}^N \rightarrow \mathbb{R}$$, we can compute the gradient $$\triangledown f = (\frac{\partial f}{\partial x_1}, \frac{\partial f}{\partial x_2},...,\frac{\partial f}{\partial x_n})$$ as

$$ \frac{\partial f}{\partial x_i} \approx \lim_{h \rightarrow 0} \frac{f(x_i + h) - f(x_i)}{h}$$

where $$h > 0$$ is some small step size. Essentially, given some tangent line between $$f(x_i)$$ and $$f(x_i + h)$$, we estimate its slope by calculating the slope of *secant line*  between $$(x, f(x))$$ and $$(x_i + h , f(x_i + h))$$, and say that as $$h$$ approaches 0, the derivative of the secant line approaches the derivative of the tangent line. Here's a graphic of that process:

<img src="https://kernelmachine.github.io/public/20170318/num_diff.png">

The issue with numerical differentiation is that it is inherently ill-conditioned and unstable. It cannot be exactly fitted to a finite representation without rounding or truncation, thus introducing approximation errors in the computation. The size of $$h$$ directly correlates with the amount of instability in the differentiation. If $$h$$ is too small, then the subtraction from $$x_i$$ will yield a larger rounding error. On the other hand, if $$h$$ is too large, the estimate of the slope of the tangent could be worse. Most people advise against using the finite differences approximation of the derivative in machine learning systems.

## Symbolic differentiation

One insight is that if we were somehow able to algebraically map functions to their derivatives, then we could likely achieve near full precision on our derivatives. For example, it is well known that the derivative of $$f(x) = x^2$$ is $$2x$$. Computing $$2x$$ is trivial, compared to using the finite differences approximation, with all its computational problems.

This is one of the primary motives of *symbolic differentiation*, the automatic algebraic manipulation of expressions based on well known rules of differentation such as:

$$ \frac{\partial}{\partial x}(f(x) + g(x)) = \frac{\partial f}{\partial x} + \frac{\partial g}{\partial x} $$

or

$$ \frac{\partial}{\partial x}(\frac{f(x)}{g(x)}) = \frac{g(x)\frac{\partial f}{\partial x} - f(x)\frac{\partial g}{\partial x}}{g(x)^2}$$

Symbolic differentiation is a representational problem, and can quickly become extremely complex, depending on the function to be manipulated. But once the representation is achieved, derivatives can be computed much more accurately than with finite differences approximation. I won't be going into more detail on symbolic differentiation's internals, mainly because it's complex, but I'll point you [here](http://homepage.divms.uiowa.edu/~stroyan/CTLC3rdEd/3rdCTLCText/Chapters/ch6.pdf) if you want more details.

## Automatic Differentiation in Forward Mode

Automatic Differentiation (AD) is a method that is both exact and simple. The basic motivation for AD is that *any arbitrary function can be represented as a composition of simpler ones*. Under functional composition, one can compute the derivative of a function using the *chain rule*:

$$ f(x) = g(h(x)) \implies f'(x) = g'(h(x)) h'(x)$$

Because $$g$$ and $$h$$ are assumed to be simpler components of the arbitrary function $$f$$, their derivatives are also simpler to compute.

An example would help explain this idea.

Let

$$f(x) = \frac{ln(x)(x+3) + x^2} {sin(x)}$$

We could take the derivative of $$f$$ using the rules we learned in high school. But there is an easier way. Fortunately, $$f$$ is a composition of very simple functions:

$$
\begin{align}
& v_0 = ln(x) \\
& v_1 = x + 3 \\
& v_2 = v_0v_1 \\
& v_3 = x^2 \\
& v_4 = v_2 + v_3 \\
& v_5 = sin(x)\\
& v_6 = \frac{v_4}{v_5}\\
& f = v_6
\end{align}
$$

Let $$\dot v_i = \frac{\partial v_i}{\partial x}$$. Differentiation proceeds via a forward pass:

$$
\begin{align}
& \dot v_0 = \frac{1}{x} \\
& \dot v_1 = 1\\
& \dot v_2 = v_0 \dot v_1 + \dot v_0 v_1 = ln(x) + \frac{1}{x}\\
& \dot v_3 = 2x \\
& \dot v_4 = \dot v_2 + \dot v_3 = ln(x) + \frac{1}{x} + 2x\\
& \dot v_5 = cos(x)\\
& \dot v_6 = \frac{v_5 \dot v_4 - v_4 \dot v_5 }{v_5^2}\\
& \dot f = \dot v_6 = \frac{sin(x)(ln(x) + \frac{1}{x} + 2x) - (ln(x)(x+3) + x^2)cos(x)}{sin^2(x)}
\end{align}
$$

The important things to notice are that:

  * The forward computation of $$\dot f$$ involves the **staged computation** of very simple components of the function. These stages are trivial to compute.

  * There is a linear flow through the derivative computation, mostly involving mere substitutions, which lends itself well to imperative programming.  

  * We compute the derivative *exactly*, and do not approximate any value.

Now, let's look at the multi-dimensional case. Let $$f : \mathbb{R}^N \rightarrow \mathbb{R}^M$$.

The derivative of $$f$$ is expressed by the Jacobian matrix

$$J_f =

\begin{bmatrix}
\partial f_1 \over \partial x_1 & ... & \partial f_1 \over x_m \\
\vdots  & \ddots & \vdots \\
\partial f_n \over \partial x_1 &  ... & \partial f_n \over \partial x_m \\
\end{bmatrix}
$$

$$J_f$$ can be computed in just $$n$$ forward AD passes across each dimension of $$f$$.

When $$ N << M $$, then the forward pass is extremely efficient in computing the Jacobian.

However, when $$ N >> M $$, another version of AD, called *reverse AD*, is more efficent to compute derivatives of $$f$$ with respect to each input dimension. We'll dig into that next.

## Automatic Differentiation in Reverse Mode

As opposed to forward autodifferentiation, which involves staged computations from the input to the output, reverse autodifferentiation evolves backwards from the function output.

We propagate the derivative of $$f$$ with the *adjoint* operator

$$\bar v_i = \frac{\partial f}{\partial v_i}$$

where $$v_i$$ is some intermediate stage in the overall function computation. These derivatives measure the sensitivity of the output of the forward pass with respect to a particular stage.

Reverse AD proceeds in two phases, one of a forward pass computation, and then reverse accumulation. The forward pass, like we outlined in the previous section, helps us keep track of the computational stages that make up the arbitrary function $$f$$.  We then compute derivatives backwards until we arrive at the original input: $$\bar x = \frac{\partial f}{\partial x}$$. Reverse AD is the preferred procedure when the function $$f : \mathbb{R}^N \rightarrow \mathbb{R}^M$$ has $$ N >> M $$ because in reverse AD we only have to make $$m$$ passes to compute the multi-dimensional gradient.

For example, if

$$\triangledown f = (\frac{\partial f}{\partial x_1}, \frac{\partial f}{\partial x_2},...,\frac{\partial f}{\partial x_n})$$

Then we only have to do 1 pass of reverse AD to compute $$\triangledown f$$, while we have to do $$N$$ passes of forward AD to get the same answer.

Let's go through an example of doing reverse AD to make the procedure clearer.

Let

$$f(x_1, x_2) = \frac{ln(x_1) + x_2^2} {sin(x_1)x_2}$$

First we do a forward pass on $$f$$

$$
\begin{align}
& v_0 = x_1\\
& v_1 = x_2\\
& v_2 = ln(v_0) \\
& v_3 = v_1^2 \\
& v_4 = v_2 + v_3 \\
& v_5 = sin(v_0)v_1\\
& v_6 = \frac{v_4}{v_5}\\
& f = v_6
\end{align}
$$


To compute $$\bar x_1 = \frac{\partial f}{\partial x_1}$$ and $$\bar x_2 = \frac{\partial f}{\partial x_2}$$, recognize that $$x_1$$ and $$x_2$$ affect the output $$f$$ in distinct ways.

In fact, it is helpful to view the forward pass as a computation graph, to visualize this point:

<img src="https://kernelmachine.github.io/public/20170318/graph.png">

$$x_1$$ only affects $$f$$ through $$v_2$$ and $$v_5$$, which means that through the multi-dimensional chain rule:

$$\frac{\partial f}{\partial x_1} = \frac{\partial f}{\partial v_0} = \frac{\partial f}{\partial v_2}\frac{\partial v_2}{\partial v_0} + \frac{\partial f}{\partial v_5}\frac{\partial v_5 }{\partial v_0} $$

or

$$ \bar v_0 = \bar v_2 \frac{\partial v_2}{\partial v_0} +  \bar v_5 \frac{\partial v_5 }{\partial v_0}$$

On the other hand, $$x_2$$ only affects $$f$$ through $$v_3$$ and $$v_5$$, so:

$$\frac{\partial f}{\partial x_2} = \frac{\partial f}{\partial v_1} = \frac{\partial f}{\partial v_3}\frac{\partial v_3}{\partial v_1} + \frac{\partial f}{\partial v_5}\frac{\partial v_5 }{\partial v_1} $$

or

$$ \bar v_1 = \bar v_3 \frac{\partial v_3}{\partial v_1} +  \bar v_5 \frac{\partial v_5 }{\partial v_1}$$

So, we can compute $$\frac{\partial f}{\partial x_1}$$ and $$\frac{\partial f}{\partial x_2}$$ with very elementary operations.

To calculate these decompositions of our gradient, we begin at the output $$f = v_6$$ and propagate its derivative backwards:

$$
\begin{align}
& \bar v_6 = \bar f\\
& \bar v_5 = \bar v_6 \frac{\partial v_6}{\partial v_5} = \bar v_6 \frac{-v_4}{v_5^2}\\
& \bar v_4 = \bar v_6 \frac{\partial v_6}{\partial v_4} = \bar v_6 \frac{1}{v_5}  \\
& \bar v_3 = \bar v_4 \frac{\partial v_4}{\partial v_3} = \bar v_4 \textbf{1} \\
& \bar v_2 = \bar v_4 \frac{\partial v_4}{\partial v_2} = \bar v_4 \textbf{1}\\
& \bar v_1 = \bar v_3 \frac{\partial v_3}{\partial v_1} = \bar v_3 2v_1\\
& \bar v_0 = \bar v_2 \frac{\partial v_2}{\partial v_0} = \bar v_2 \frac{1}{v_0}\\
& \bar x_2 = \bar v_1\\
& \bar x_1 = \bar v_0\\
\end{align}
$$

By keeping track of stages in the forward pass of the reverse AD, the bottleneck of computing $$\frac{\partial f}{\partial x_1}$$ and  $$\frac{\partial f}{\partial x_2}$$ is reduced to computing $$\bar v_6$$, which is only dependent on the complexity of the final stage of computation in $$f$$. As an exercise, set $$\bar v_6 = 1$$ and compute the gradient of $$f$$.

Reverse autodifferentation is known as *backpropagation* in deep learning, and forms the basic way that we update parameters of a neural network during learning. In the next section, we'll dive into how to apply reverse autodifferentation to train neural networks.

## Backpropagation in Deep Learning

Consider a two layer neural network. Given an input matrix $$X \in R^{N x M}$$ and output labels $$y \in R^N$$, let $$W_1$$ and $$W_2$$ be weight matrices corresponding to layer 1 and 2 respectively, and $$b_1$$ and $$b_2$$ be bias vectors corresponding to each weight matrix. Between layer 1 and 2 imagine we have an ReLU activation function $$f(x) = max(0, x)$$, and imagine that we apply a softmax classifier after layer 2 to squash its output between $$[0, 1]$$.

Here's a rough diagram of the network we'll be working with.

<img src="https://kernelmachine.github.io/public/20170318/nnet.png">

During learning, we want to provide input to the neural network, and then update the weights at each layer depending on the error computed by the loss function at the output. We'll use reverse AD (or *backpropagation*) to find the gradients of the loss function with respect to each weight matrix. Note that all the layers can be updated by merely knowing the derivative of the *last stage of computation*, because as we saw in the last section, all previous stages' derivatives can then be computed. This means that we'll have to figure out what the derivative of our the softmax loss function is, and then we're golden.

We can model the neural network as a forward pass of staged computations from the input $$X$$ to the softmax loss function:

$$
\begin{align}
& Y_1 = W_1X + b_1 \\
& A_1 = max(0, Y_1) \\
& Y_2 = W_2A_1 + b_2 \\
& L = SoftmaxLoss(Y_2)\\
\end{align}
$$

Here's what the forward pass (prior to the loss function) would look like in Python, computing the class scores for the input.

```python
import numpy  as np

Y1 = X.dot(W1) + b1 # first layer
A1 = np.maximum(0, Y1) #  ReLU activation
Y2 = A1.dot(W2)+ b2 # second layer
scores = np.exp(Y2) / np.sum(np.exp(Y2), axis=1, keepdims=True) # softmax
```

How do we define the $$SoftmaxLoss$$ function in the output?

Well, the softmax function applied to a score vector $$f$$ is

$$
p_k = \frac{e^{f_k}}{\sum_j e^{f_j}}
$$

The data loss function for a softmax classifier is

$$ L_i = -log(p_{y_i})$$

Where $$y_i$$ is the index of the correct label.

The softmax classifier loss for this network is defined as

$$
L = \underbrace{\frac{1}{N}\sum_i L_i}_\text{data loss} + \underbrace{\lambda [\sum_i\sum_j W_1^2 + \sum_i\sum_j W_2^2]}_\text{regularization loss}
$$

Rounding out the code for the forward pass, here's what the loss function looks like in Python:

```python
import numpy  as np

data_loss = -np.log(scores[range(N), y])
reg_loss = 0.5 * reg * np.sum(W1 * W1)  + 0.5 * reg * np.sum(W2 * W2)
loss = np.sum(data_loss) / N + reg_loss
```

During learning, we use gradient descent to optimize the network's weight matrices $$W_1$$ and $$W_2$$. We update the weights with their gradients on $$L$$ , $$\partial L \over \partial W_1$$ and $$\partial L \over \partial W_2 $$.

To find $$\partial L \over \partial W_1$$ and $$\partial L \over \partial W_2 $$, we do the reverse accumulation phase of backpropagation.


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

We want to find $$\frac{\partial L}{\partial Y_2}$$. $$Y_2$$ is just an unnormalized score vector. In the following equations, we will set an alias $$f  = Y_2$$ for clarity. Consider first the data loss, $$ H_j = \frac{1}{N}\sum_i L_i = -\frac{1}{N}\sum_i log(p_j)$$.

When $$ j = y_i $$,

$$ \frac{\partial H}{\partial f_{y_i}} = -\frac{1}{N} \sum_i \frac{\partial}{\partial f_{y_i}} log(p_{y_i}) \frac{\partial p_{y_i}}{\partial f_{y_i}}$$

$$  = -\frac{1}{N} \sum_i \frac{1}{p_{y_i}}  \frac{\sum_j e^{f_{j}} e^{f_{y_i}} - e^{f_{y_i}} e^{f_{y_i}} } {\sum_j e^{f_j}}$$

$$  = -\frac{1}{N} \sum_i \frac{1}{p_{y_i}} \frac{e^{f_{y_i}} (\sum_j e^{f_{j}}  - e^{f_{y_i}})}{\sum_j e^{f_j}}$$

$$  = -\frac{1}{N} \sum_i \frac{1}{p_{y_i}} p_{y_i}(1 - p_{y_i})$$

$$  = p_{y_i} - 1$$



When $$ j \neq y_i $$,

$$ \frac{\partial H}{\partial f_{j}} = -\frac{1}{N} \sum_i \frac{\partial}{\partial f_{j}} log(p_{y_i}) \frac{\partial p_{y_i}}{\partial f_{j}}$$

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

Where $$\mathbb{1}$$ is just a binary indicator function that evaluates to 1 when the predicate is true.

Now we can code the backpropagation onto the softmax loss function:

```python
# backprop onto loss function
dscores = scores
dscores[range(N),y] -= 1
dscores /= N
```

Taking into account the gradient of the regularization term in $$L$$,

$$
\triangledown L = \frac{\triangledown H}{N} + \lambda\sum\limits_{i}\sum\limits_{j}W_{ij}
$$

We'll add the regularization gradient at the end of the backpropagation.

With the gradient of the loss function, we can easily calculate $$ \bar W_2 $$ and $$\bar b_2$$

```python
grads = {}
# backprop onto W2 and b2
grads['W2'] = a1.T.dot(dscores)
grads['b2'] = np.sum(dscores, axis = 0)
```

Now we have to compute $$\bar W_1$$.

$$\bar W_2 = \bar Y_1 X  = \bar A_1 \frac{\partial A_1}{\partial Y_1} X = \bar Y_2 W_2 \frac{\partial A_1}{\partial Y_1} X $$

To compute $$\bar W_1$$, we first have to backpropagate onto the ReLU activation function. In other words, we have to find $$\frac{\partial A_1}{\partial Y_1}$$:

$$ A_1 = max(0, Y_1) \implies \frac{\partial A_1}{\partial Y_1}=\mathbb{1}( Y_1 >0)$$

Combined with the chain rule, we see that the ReLU unit lets the gradient pass through unchanged if its input was greater than 0, but kills it if its input was less than zero during the forward pass.

```python
# backprop onto ReLU activation
dhidden = dscores.dot(W2.T)
dhidden[a1 <= 0] = 0
```

Now we can calculate $$ \bar W_1 $$ and $$\bar b_1$$:

```python
# backprop onto W1 and b2
grads['W1'] = X.T.dot(dhidden)
grads['b1'] = np.sum(dhidden, axis = 0)
```

Finally, remember to add in the backpropagation onto the regularization loss:

```python
# don't forget regularization loss
grads['W2'] += reg * W2
grads['W1'] += reg * W1
```

Here's the full code to compute the backpropagation on our network:

```python
import numpy  as np

## forward pass
Y1 = X.dot(W1) + b1 # first layer
A1 = np.maximum(0, Y1) #  ReLU activation
Y2 = A1.dot(W2)+ b2 # second layer
scores = np.exp(Y2) / np.sum(np.exp(Y2), axis=1, keepdims=True) # softmax

## loss function
data_loss = -np.log(scores[range(N), y])
reg_loss = 0.5 * reg * np.sum(W1 * W1)  + 0.5 * reg * np.sum(W2 * W2)
loss = np.sum(data_loss) / N + reg_loss

grads = {}

# backprop onto loss function
dscores = scores
dscores[range(N),y] -= 1
dscores /= N

# backprop onto W2 and b2
grads['W2'] = a1.T.dot(dscores)
grads['b2'] = np.sum(dscores, axis = 0)

# backprop onto ReLU activation
dhidden = dscores.dot(W2.T)
dhidden[a1 <= 0] = 0

# backprop onto W1 and b2
grads['W1'] = X.T.dot(dhidden)
grads['b1'] = np.sum(dhidden, axis = 0)

# don't forget regularization loss
grads['W2'] += reg * W2
grads['W1'] += reg * W1
```

We would use these gradients to then update $$W_1$$ and $$W_2$$ with gradient descent.
