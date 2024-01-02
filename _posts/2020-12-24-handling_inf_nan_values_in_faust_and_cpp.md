---
layout: post
title: Handling infinity and not-a-number (NaN) values in Faust and C++ audio programming
date: 2020-12-28
description: This post discusses insights gained over a few years of audio programming to implement robust Faust/C++ software, particularly when dealing with infinity and NaN values.
categories: Posts
tags: audio-programming
---

Faust is a high-level programming language for audio based on C++ where several implementation-related low-level issues are handled by the Faust compiler. However, Faust still allows for a certain degree of freedom in its operations that may result in undesired behaviours for the domain of audio streams. This blog post shows a few examples where the special values of $ \mathit{inf} $ and $ NaN $ can be generated and how to avoid that. As we will see, $ \mathit{inf} $ and $ NaN $ values are problematic for audio streams as they follow peculiar arithmetic and relational rules.

$ \mathit{Inf} $, both positive or negative, is a special value that is used to represent an exceedingly large number, namely a value that is too large to be represented accurately by the floating-point type used. For example, exponential and unbounded growths can occur in unstable recursive systems, which eventually result in $ \mathit{inf} $ or $ \mathit{-inf} $. Also, in recursive systems, the opposite case, that is an exponential decay where a value decreases and tends towards 0, can still be a problem when reaching the representation limits of the data type for small values. These values are commonly called *subnormal* values, and they can be CPU-intensive. Luckily, efficient *flush-to-zero* (FTZ) mechanisms are usually deployed at the hardware-level to overcome the problem.

$ \mathit{Inf} $ or $ \mathit{-inf} $ can also be generated through the multiplication of very large values, or by the division between a large value and a small enough one. Below, we can summarise the arithmetic of $ \mathit{inf} $ values, alone or combined with real numbers, through the output of common operators and functions in Faust and C++. These examples also show how [indeterminate forms](https://en.wikipedia.org/wiki/Indeterminate_form) are handled.

$$
\begin{align}
& \infty \cdot x = \infty \cdot sgn(x) \quad \{x \in \mathbb{R} : x \neq 0\} \\ & \infty \cdot (\pm \infty) = \pm \infty \\ & \pm \infty \cdot 0 = NaN \\ & \infty / x = \infty \cdot sgn(x) \quad \{x \in \mathbb{R} : x \neq 0\} \\ & \pm \infty / (\pm \infty) = NaN \\ & \pm \infty / 0 = \pm \infty \\ & 0 / 0 = NaN \\ & x \bmod 0 = NaN \quad \{x \in \mathbb{R}\} \\ & \pm \infty \bmod x = NaN \quad \{x \in \mathbb{R}\} \\ & \pm \infty + x = \pm \infty \quad \{x \in \mathbb{R}\} \\ & \pm \infty - x = \pm \infty \quad \{x \in \mathbb{R}\} \\ & \pm \infty \pm \infty = \pm \infty \\ & \infty - \infty = NaN \\ & \pm \infty^0 = 1 \\ & \pm \infty^1 = \pm \infty \\ & \pm \infty^{-1} = \pm 0 \\ & \pm \infty^{\infty} = \infty \\ & \pm \infty^{-\infty} = 0 \\ & \pm 1^{\pm \infty} = 1 \\ & 0^0 = 1 \\ & \sqrt{x} = NaN \quad \{x \in \mathbb{R} : x < 0\} \\ & \sqrt{\infty} = \infty \\ & \sqrt{-\infty} = NaN \\ & \log(0) = -\infty \\ & \log(\infty) = \infty \\ & \log(x) = NaN \quad \{x \in \mathbb{R} : x < 0\} \\ & cos(\pm \infty) = NaN \\ & sin(\pm \infty) = NaN \\ & tan(\pm \infty) = NaN \\ & acos(\pm \infty) = NaN \\ & acos(x) = NaN \quad \{x \in \mathbb{R} : x < -1 \lor x > 1\} \\ & asin(\pm \infty) = NaN \\ & asin(x) = NaN \quad \{x \in \mathbb{R} : x < -1 \lor x > 1\} \\ & atan(\pm \infty) = \pm \pi / 2 \\ & acosh(x) = NaN \quad \{x \in \mathbb{R} : x < 1\} \\ & acosh(\infty) = \infty \\ & acosh(-\infty) = NaN \\ & asinh(\pm \infty) = \pm \infty \\ & atanh(\pm 1) = \pm \infty \\ & atanh(x) = NaN \quad \{x \in \mathbb{R} : x < -1 \lor x > 1\} \\ & cosh(\pm \infty) = \infty \\ & sinh(\pm \infty) = \pm \infty \\ & tanh(\pm \infty) = \pm 1 \\
\end{align}
$$

We can see that several of these operations produce $ NaN $, which is an even more problematic value for audio. Particularly, *any* operation where one of the operands is $ NaN $ produces a $ NaN $, and any relational operation containing a $ NaN $ is false. Thus, the possibility of a $ NaN $ value contaminating the audio stream may result in a chain reaction where these values spread really fast. If a $ NaN $ value is used to access an array cell, for example, in a delay line, the program is likely to end with a *segmentation fault* error. Thus, it is vital to prevent $ NaNs $ to enter audio streams, as well as to prevent $ \mathit{inf} $ values since they, too, can result in $ NaNs $. Also, note that signed zeroes are important for FTZ mechanisms, as we can at least preserve the sign of the subnormal value. See [[Goldberg 1991]](https://www.validlab.com/goldberg/paper.pdf) for more.

C++ provides useful values for representable limits in the *limits* [library](http://www.cplusplus.com/reference/limits/numeric_limits/). The *std::numeric_limits::max()*, *std::numeric_limits::min()*, and *std::numeric_limits::epsilon()* functions output constants representing, respectively, the largest and smallest representable values, and the relative rounding error such that $ \epsilon $ is the smallest quantity for which $ 1 + \epsilon > 1$ returns $ true &. In double precision, these are:

$$
\begin{align}
& 1.7976931348623158e+308 \\ & 2.2250738585072014e-308 \\ & 2.2204460492503131e-016
\end{align}
$$

The constant $ \epsilon $ can be useful when some variable must be set to a value just below $ 1 $, for example, when we need the pole of a filter as close as possible to the unit circle. The constant $ MIN $ can be used as a limit to avoid division by $ 0 $. For example, a safe division operator in Faust can be implemented as follows:

{% highlight faust linenos %}

import("stdfaust.lib");
safe_div(x, y) = ba.if(y < 0, x / min(ma.MIN * -1, y), x / max(ma.MIN, y));
process = safe_div(1, 0);

{% endhighlight %}

The output of the division $ 1/0 $ using the *safe_div* function is $ 4.4942328371557898e+307 $. It is very close to the $ MAX $ constant, meaning that it can easily become $ \mathit{inf} $. Alternatively, the $ \epsilon $ constant can be used as a limit as in:

{% highlight faust linenos %}

import("stdfaust.lib");
safe_div(x, y) = ba.if(y < 0, x / min(ma.EPSILON * -1, y), x / max(ma.EPSILON, y));
process = safe_div(1, 0);

{% endhighlight %}

In this case, the output of $ 1/0 $ is $ 4503599627370496 $, which is in a much safer range at the expenses of accuracy for some divisions. Still, if the numerator were greater than $ MAX \cdot \epsilon $, it would become $ \mathit{inf} $. Otherwise, we could simply clip the output of the $ / $ operator to $ MAX $ and $ -MAX $ to guarantee that neither $ \mathit{inf} $ or $ NaN $ values are output:

{% highlight faust linenos %}

import("stdfaust.lib");
den = (ma.INFINITY' ^ 2 * -1) ^ -1;
max_clip(x) = max(ma.INFINITY * -1, min(ma.INFINITY, x));
safe_div(x, y) = max_clip(x / y);
process = safe_div(1, den);

{% endhighlight %}

Note that $ \mathit{ma.INFINITY} $ corresponds to $ MAX $ in Faust, and that we must delay the value that defines the denominator; otherwise, the Faust compiler will detect a division by zero and will fail to compile. For the denominator, we generated a $$ -0 $$ at the second sample, which results in the $ -MAX $ constant. Also note that this division is effective to keep audio streams clean from $ NaN $ and $ \mathit{inf} $ values, although the division by zero can still take place. However, if the hardware follows the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) standard, the floating-point division by zero will produce $ \pm \mathit{inf} $ instead of being signalled as an exception by the C++ program, unless both numerator and denominator are $ 0 $, in which case it will produce $ NaN $. For the same reason, consider that Faust's syntax is strict and that all branches of an if-statement are always evaluated; you may want to read [this](https://github.com/grame-cncm/faustdoc/blob/master/mkdocs/docs/manual/faq.md) as well. 

Another problem of clipping only the output of a function as a guard is that $ NaN $ values are never output in *std::max()* and *std::min()* functions, hence whether the indeterminate form $ 0/0 $ results in $ MAX $ or $ MIN $ only depends on the implementation of the clipping function, namely whether we first check against the upper or lower limit. In general, the most effective guards that we have against $ NaN $ and $ \mathit{inf} $ values are the *std::max()* and *std::min()* functions combined with the numerical limit constants above to effectively limit the domain of other functions. If an operator or function is indeterminate for some input, then it is necessary to limit the input domain and, in some cases, the output domain too. Otherwise, limiting only the output is adequate. A strict safe division function in Faust could then look like this:

{% highlight faust linenos %}

import("stdfaust.lib");
max_clip(x) = max(ma.INFINITY * -1, min(ma.INFINITY, x));
safe_div(x, y) = 
    max_clip(ba.if(y < 0, x / min(ma.EPSILON * -1, y), x / max(ma.EPSILON, y)));
process = safe_div(0, 0);

{% endhighlight %}

In this case, the output of $ 0 $ divided by any real value including $ 0 $ is always $ 0 $. A safe *std::sinh()* function, instead, can be defined as follows, which will guarantee real values between  $ MAX $ and $ -MAX $ just by clipping its output:

{% highlight faust linenos %}

import("stdfaust.lib");
max_clip(x) = max(ma.INFINITY * -1, min(ma.INFINITY, x));
safe_sinh(x) = max_clip(ma.sinh(x));
process = safe_sinh(ma.INFINITY);

{% endhighlight %}

For the *std::log()* function, for example, the input domain could be limited to $ MIN $ and $ MAX $, giving a rather safe output domain between $ -708.39641853226408 $ and $ 709.78271289338397 $:

{% highlight faust linenos %}

import("stdfaust.lib");
safe_log(x) = log(max(ma.MIN, min(ma.INFINITY, x)));
process =   safe_log(0) ,
            safe_log(ma.INFINITY^2);

{% endhighlight %}

Alternatively, the output domain can be clipped to $ -MAX $ and $ MAX $, making sure that we first check against the lower bound so that $ NaN $ values generated by negative inputs are clipped to $ -MAX $:

{% highlight faust linenos %}

import("stdfaust.lib");
safe_log(x) = min(ma.INFINITY, max(ma.INFINITY * -1, log(x)));
process =   safe_log(0) ,
            safe_log(ma.INFINITY^2);

{% endhighlight %}

Finally, I would like to thank my brother, Salvatore Sanfilippo, and Oli Larkin for a few valuable comments on this post.
