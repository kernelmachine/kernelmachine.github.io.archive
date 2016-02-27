---
layout: post
title: Out-of-Core Learning
--

The cloud has given machine learning practitioners unprecedented strength to build and test models with less concern about memory and CPU usage. Amazon will be introducing X1 instances this year, with 2 TB of memory [^1].

But powerful instances are expensive, and non-trivial to set up.

So ideally we'd want to use the smallest instance possible, but a lack of memory can spell the death of machine learning models. Say you have a dataframe of 1 million samples with 20 `float64` features each. A `float64` element takes 8 bytes. This dataframe will have a size of:

$$ 10e^6 \cdot 20 \cdot 8 = 160,000,000 \text{ bytes} ~= 160 \text{ gigabytes!} $$

The smallest EC2 instance we could use for this data is BLAH, which would cost BLAH.

What are some solutions?

### Out-of-Core learning

{% highlight python %}

import pandas as pd
urls = ['http://google.com']
pd.DataFrame({'url':urls})

{% endhighlight %}

when $$x + y$$ we have


[^1]: [Lardinois, Frederic. "AWS Announces X1 Instances For EC2 With 2TB Of Memory, Launching Next Year." TechCrunch, 8 Oct. 2015.](http://techcrunch.com/2015/10/08/aws-announces-x1-instances-for-ec2-with-2tb-of-memory-launching-next-year/)
