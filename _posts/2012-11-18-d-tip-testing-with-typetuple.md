---
layout: post
title: D Tip - Testing with TypeTuple
tags:
- d
- dlang
- tip
- typetuple
- testing
- unittesting
- metaprogramming
---
When testing a function, it's usually a good idea to test it with a range
of different inputs. To do this, you can easily use a `for` loop over an array
of input values, but what if your input is a _type_, as it often is with
template code?

The [D Programming Language][1] allows you to iterate over a `TypeTuple`, so
all you need to do is declare a tuple of all the types you want to test, and
iterate over them in the normal way:

{% highlight d %}
import std.typetuple;
alias TypeTuple!(int, long, double) Types;
foreach (T; Types)
    test!T();
{% endhighlight %}

You might wonder what this compiles to. After all, the body of the loop varies
with `T`, so the generated code must also vary on each iteration. How does the
compiler handle this?

The answer is that the loop is completely unrolled. The code above is literally
the same as:

{% highlight d %}
test!int();
test!long();
test!double();
{% endhighlight %}

For this reason, you might want to keep an eye on the size of your `TypeTuple`s,
to avoid code bloat.

[1]: http://dlang.org