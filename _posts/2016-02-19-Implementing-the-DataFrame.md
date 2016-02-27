---
layout: post
title: Implementing the DataFrame in Rust
comments: true
---

The pandas dataframe is an annoying data structure from a type perspective. It's a structure
in which floats, ints, and strings can co-exist.

{% highlight bash %}

>> np.sctypes
{'complex': [numpy.complex64,
            numpy.complex128,
            numpy.complex192],
'float':   [numpy.float16,
           numpy.float32,
           numpy.float64,
           numpy.float96],
'int':     [numpy.int8,
            numpy.int16,
            numpy.int32,
            numpy.int32,
            numpy.int64],
 'others': [bool,
            object,
            str,
            unicode,
            numpy.void],
 'uint': [numpy.uint8,
          numpy.uint16,
          numpy.uint32,
          numpy.uint32,
          numpy.uint64]
        }

{% endhighlight %}
