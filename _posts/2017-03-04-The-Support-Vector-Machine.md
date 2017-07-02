---
layout: post
title: Crash Course on Support Vector Machines
comments: true
---

The support vector machine (SVM) is one of the basic linear classifiers. This post will breakdown how the vanilla algorithm is implemented.

## Basic approach


Let's first get some preliminaries out of the way. Let $$\{x_1,...,x_n\} \in X$$ be a dataset describing our classification problem, and $$\{w_1,...,w_n\} \in W$$ be a weight matrix that we want to optimize. The SVM takes the data in $$X \in \mathbb{R}^N$$ and weights in $$W \in \mathbb{R}^N$$ to compute the vectors $$W^TX$$ of scores for each data sample per class. In other words, the value $$w^T_jx_i$$ denotes the score for j-th class for the i-th data sample. Suppose we are futher given the labels $$\{y_1,...,y_n\}$$ that specifies the *index* of the correct class for each data sample. For example, $$w^T_{y_i}x_i$$ is the score for the *true* class for the i-th data sample.

The basic problem of the support vector machine is to find the best *hyperplane(s)*, or subspace(s) of one less dimension than the ambient space, that best separate data points into distinct classes.

How does one identify the optimal hyperplane? Well, every hyperplane can be written in the form:

$$ w^Tx_i=\text{b} \implies w^Tx_i - b = 0 $$

Where $$w^Tx_i$$ is some vector translated by the norm of $$b$$, called the bias term.

The SVM amounts to finding hyperplanes such that

$$w^T_{j}x_i - w^T_{y_i}x_i  - b  \ge \delta \text{   } \forall  \text{   }  j \neq y_i $$

$$\delta$$ is called the *margin*, and corresponds to the euclidean distance between the separating hyperplanes. We essentially don't want data points to fall inside the margin, and the optimal decision boundary lies at the midpoint of the margin.

 This is an example of a *soft margin* SVM. There is also a *hard margin* version, in which the margin is completely determined by the closest data-points to the decision boundary,  called the *support vectors*.


<img src="http://kernelmachine.github.io/public/20170304/svm.png">


Note that it can be shown that the canonical binary classification SVM problem is just a special case of the multi-class formulation above. In particular, it can be shown that in the binary case the above formulation reduces to

$$ sgn(w^Tx_i - b) \ge \delta$$



Finally, note that from here on out we will disregard the bias term $$b$$ in the above formulation, and will assume that the bias term is incorporated into the weight matrix $$W$$ as an extra column.

## Loss Function

Now that we've defined the general approach to classifying data with the SVM, we have to derive the loss function for the problem, which enforces how weights $$W$$ get updated during learning.


The SVM aims to minimize the following loss function $$L_i$$ for each data sample $$x_i \in X$$:

$$ L_i = \sum\limits_{j \neq y_i} max (0, w^T_{j}x_i - w^T_{y_i}x_i + \delta) $$

Essentially, we're saying that we incur loss if the difference between the scores of the correct class and those of the incorrect classes are within some margin of each other. However, once that difference passes the threshold of the margin, we always incur zero loss.

The loss function scales with:
  1) the size of the margin and
  2) how much smaller the score at the index of the incorrect class is to the score at the index of the correct class.

This implies that the SVM's output are  uncalibrated scores that roughly relate to *center of mass* of the score vector. We want the score vector for the i-th data sample to be highly concentrated at the index of the correct class.

## Regularization

We enforce preference over particular weights during optimization through *regularization*, an extra term that we add to the loss function above:

$$ R(W) = \lambda \sum\limits_{j}\sum\limits_{i}W_{ij}^2 $$

Because we are using the $$L_2$$-norm (i.e. the global sum of the squared matrix), $$R(W)$$ is called $$L_2$$ regularization. The $$\lambda$$ term is called the *regularization constant*, which is a hyperparameter we can optimize empirically.


The loss function for each data sample now looks like this:

$$ L_i = \sum\limits_{j \neq y_i} max (0, w^T_{j}x_i - w^T_{y_i}x_i + \delta) + \lambda\sum\limits_{j}\sum\limits_{i}W_{ij}^2$$

By adding the regularization term to the loss function, we are saying that we prefer score vectors that have a lower $$L_2$$-norm, which  corresponds to scores that are *lower and diffuse*. Lower, diffuse scores improve model generalization because a single dimension doesn't have overwhelming contribution to the overall prediction.

<img src="http://kernelmachine.github.io/public/20170304/reg.png">

The final loss function for the SVM is just the average loss over *all* data samples:

$$ L = \frac{1}{N}\sum\limits_iL_i + R(W) = \frac{1}{N}\sum\limits_i\sum\limits_{j \neq y_i} max (0, w^T_{j}x_i - w^T_{y_i}x_i + \delta) + \lambda\sum\limits_{i}\sum\limits_{j}W_{ij}^2$$

Here's the code for the SVM loss function in Python:

```python
import numpy as np

def svm_loss(X, y, W, delta, reg):

  ## number training samples
  num_train = X.shape[0]

  ## get scores
  scores = W.dot(X)

  # calculate margins
  margins = np.maximum(0, scores - scores[y, :] + delta)

  # margins at the index of the correct class should be zero
  margins[y, :] = 0

  # calculate l2 regularization
  l2_reg = reg * np.sum(W * W)

  # average over all samples
  data_loss = np.sum(margins) / num_train

  L = data_loss + l2_reg

  return L, margins
```

## Optimization step  

Now that we have defined the loss function for the SVM, how do we update the weights $$W$$ to minimize loss?

First we calculate the gradient of $$L$$ with respect to our weights $$W$$. Then, we update our weights to descend along our loss function in the steepest direction. We can scale how large of a step size we take along the gradient with a *learning rate* parameter.

With respect to the SVM, we first take the derivative of its loss function, which involves taking the derivative of the $$max(0, -)$$ function. That may sound scary, but it's actually pretty easy once you split out the possible ranges of the function:

$$
H_i = \sum\limits_{j \neq y_i}max (0, w_{j}^Tx_i- w_{y_i}^Tx_i + \delta) = \left\{\begin{aligned}
& 0 &&: \text{if score outside margin} \\
&w^T_{j}x_i - w^T_{y_i}x_i + \delta &&:  \text{if score inside margin}
\end{aligned}
\right.
$$

Let's look at a simple scenario to see what the derivative would look like, and generalize from there.

Suppose we had a three dimensional weight vector $$\{w_1,w_2,w_3\}$$ that we wanted to optimize with respect to the data sample $$x_i$$, and let's suppose that of the three classes $$x_i$$ could map to, the 2nd class was the true class. $$H_i$$ would then expand to:

$$
 H_i = \sum\limits_{j \neq y_i} max (0, w^T_{j}x_i - w^T_{y_i}x_i + \delta)\\
 = max (0, w^T_{1}x_i - w^T_{2}x_i + \delta) + max (0, w^T_{3}x_i - w^T_{2}x_i + \delta)\\
 = [w^T_{1}x_i - w^T_{2}x_i + \delta > 0] (w^T_{1}x_i - w^T_{2}x_i + \delta)  + [w^T_{3}x_i - w^T_{2}x_i + \delta > 0] (w^T_{3}x_i - w^T_{2}x_i + \delta)
$$

Then we take the derivative of $$L_i$$ with respect to $$w_2$$ and $$\{w_1, w_3\}$$:

$$
\frac{dH_i}{dw_{2}} = -x([w^T_{1}x_i - w^T_{2}x_i + \delta > 0] + [w^T_{3}x_i - w^T_{2}x_i + \delta > 0])\\
\frac{dH_i}{dw_{1}} = x([w^T_{1}x_i - w^T_{2}x_i + \delta > 0])\\
\frac{dH_i}{dw_{3}} = x([w^T_{3}x_i - w^T_{2}x_i + \delta > 0])
$$

Not so bad!

We can generalize the above procedure as follows:

$$
  \frac{dH_i}{dW_{y_i}} = -x(\sum\limits_{j \neq y_i}[w^Tx_j - w_{y_i}^Tx + \delta > 0])\\
  \frac{dH_i}{dW_{j}} =x(w^T_jx - w_{y_i}^Tx + \delta > 0)

$$

where $$ j \neq y_i$$.

In simple words, the gradient amounts to counting the number of times an class prediction falls within the margins of our decision boundary, scaled by $$X$$.

In its entirety, the gradient of the loss function is the average gradient across all samples, also taking into account the regularization term:

$$ \triangledown L = \frac{1}{N}\sum\limits_i[\frac{dH_i}{dW_{y_i}} + \frac{dH_i}{dW_{j}}] + \lambda\sum\limits_{i}\sum\limits_{j}W_{ij} $$


Here's the code for computing the gradient for the SVM:

```python
import numpy as np

def compute_gradient(X, y, W, margins, delta, reg):

  ## get number of training samples
  num_train = X.shape[0]

  ## compute number of errors
  num_errors = np.sum(margins > 0, axis = 0)

  ## compute derivative
  dH = np.zeros(W.shape)

  ## for dH/dWj where j != y
  dH[margins > 0] = 1

  ## for DH/dWy
  dH[y, :] = -num_errors

  dLdW = dH * X / num_train + reg * np.sum(W)

  return dLdW
```

To do the weight update, we perform the simple procedure:

```python
import numpy as np

def loss(X,y, W, delta, reg, tol):

    ## compute loss and margins
    L, margins = svm_loss(X, y, W, delta, reg)

    ## compute gradient
    dLdW = compute_gradient(X, y, W, margins, delta, reg)

    return L, dLdW

def update_weights(X, y, W, learning_rate, delta, reg, tol):

    ## compute loss and gradient
    L, dLdW = loss(X, y, W, delta, reg, tol)

    ## update weights
    W += -learning_rate * dLdW

    return L, W
```

Finally, we can create a vanilla SVM training function by initializing $$W$$ with random values:

```python
import numpy as np

def train(X, y, tol, delta, reg, learning_rate, max_iter):

  ## get appropriate dimensions
  dim, num_train = X.shape[1]
  num_classes = np.max(y) + 1

  ## initialize W with random values
  W = np.random.randn(num_classes, dim)

  ## repeat update
  curr = 0
  while L > tol and curr < max_iter:
    L, W = update_weights(X, y, W, learning_rate, delta, reg, tol)
    curr += 1

  return W

```

As a side note, in the real world, instead of computing the gradient over the *entire* training dataset, we compute gradients over subsamples. This is just more efficient for large scale applications in which the entire dataset isn't necessary to make a single step along the gradient of the loss function.

## Advantages and Issues

As with all other learning algorithms, the SVM has strengths and weaknesses.

First off, one big issue with the SVM is that its scores can be difficult to interpret. Because they are uncalibrated scores, it's hard to compare them.

Futhermore, in principle SVMs should be highly resistant to over-fitting, due to things like regularization and margins, but in practice this depends on the careful choice of these hyperparameters, which is pretty nontrivial.

However, the SVM (especially the hard-margin version) is more memory efficient than other algorithms because it primarily optimizes the classifier based on euclidean distance between support vectors, a small subset of the data. Furthermore, if a data point is outside of the margin, loss is zero no matter what. That means that the algorithm is less sensitive to outliers in the dataset.

Finally, for simplicity we have only considered the linear SVM here, but there are a multitude of non-linear and kernel-based SVMs that can fit quite intricate decision boundaries to the data. In fact, the SVM is pretty versatile once kernels are introduced.
