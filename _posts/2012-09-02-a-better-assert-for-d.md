---
layout: post
title: A Better Assert for D
tags:
- d
- assert
- programming
---
Like C and C++, the D programming language provides an `assert` mechanism
that allows you to check conditions at runtime, and halt execution if the
condition fails. You can optionally print out a error if you're in a
particularly good mood.

{% highlight d %}
float log(float x)
{
    assert(x > 0.0f, "Argument to log must be positive.");
    ...
}
{% endhighlight %}

Trying to call `log(-1)` will give you an error at runtime.

{% highlight d %}
core.exception.AssertError@test.d(3): Argument to log must be positive.
{% endhighlight %}

In this example, I know that the argument was `-1`, but what if the argument was
the result of some long calculation? It would be nice to know what was actually
passed into `log`. Typically, D programmers will use `std.string.format` for
this purpose:

{% highlight d %}
float log(float x)
{
    import std.string : format;
    assert(x > 0.0f, format("Argument must be positive (x = %f).", x));
    ...
}
{% endhighlight %}

This solves the problem, but has a couple of issues:

1. You need to import `std.string.format`.
2. The call to `format` is a bit ugly.

Granted, these are relatively minor issues, but it would be nice if we could
simply write:

{% highlight d %}
float log(float x)
{
    assert(x > 0.0f, "Argument must be positive (x = %f).", x);
    ...
}
{% endhighlight %}

C and C++ programmers typically work around this by using some clever macros,
but D doesn't have a pre-processor, so we'll need to use standard functions.

Here's a first attempt:

{% highlight d %}
void assertf(Args...)(bool test, Args args)
{
    import std.string : format;
    assert(test, format(args));
}

float log(float x)
{
    assertf(x > 0.0f, "Argument must be positive (x = %f).", x);
    ...
}
{% endhighlight %}

This works, giving the error:

{% highlight d %}
core.exception.AssertError@test.d(4): Argument must be positive (x = -1).
{% endhighlight %}

Notice however, that the file and line number are of the assert inside
`assertf` rather than `log`. This is less than ideal.

Fortunately, D has a tricky little feature designed just for this. D defines
a couple of keywords, `__FILE__` and `__LINE__` that statically evaluate to the
file name, and line number of the locations of those keywords respectively.
Furthermore, if you use those keywords as default arguments to a function then
you get the file name and line number of the calling function. For example:

{% highlight d %}
void getInfo(string file = __FILE__, int line = __LINE__)
{
    import std.stdio;
    writeln(file, " : ", line);
}

// test.d
void main()
{
    getInfo(); // test.d : 10
    getInfo(); // test.d : 11
}
{% endhighlight %}

This is exactly what we need. There's just one problem: you can't put default
arguments after variadic parameters; the compiler can't figure out whether
the last arguments refer to the variadic parameters or the optional parameters.

The way to work around this is to make the file and line *template* parameters,
that way they can be automatically deduced, and the user won't need to specify
them manually.

{% highlight d %}
void assertf(string file = __FILE__, int line = __LINE__, Args...)
            (bool test, Args args)
{
    import std.string : format;
    import core.exception : AssertError;
    if (!test)
    {
    	throw new AssertError(format(args), file, line);
    }
}
{% endhighlight %}

Our function is working as expected now, but suffers a major, obvious flaw: it
works in `-release` builds.

While D provides a standard way to detect `-debug` builds (using the `debug`
keyword), there is *currently* no standard way to detect builds where an assert
should fire.

I say "*currently*" because I have requested the feature, and as of
DMD 2.061, you'll be able to write:

{% highlight d %}
version(assert) if (!test)
{
    throw new AssertError(format(args), file, line);
}
{% endhighlight %}

This is conditionally compile the test only in builds where asserts are meant
to fire, similar to how we can currently use `version(unittest)` to detect
unit testing builds.

The final remaining difference between our function and the built-in assert
is the [evaluation strategy][eval]. When asserts are compiled out of your
build, not only does the assert do nothing, but the arguments aren't even
evaluated. For example:

{% highlight d %}
assert(printf("Hello, world") > 0);
{% endhighlight %}

In `-release` builds, this will print out nothing, however the analgous call
to our formatting assert will still print out.

The way to work around this is to use [lazy evaluation][lazy].

{% highlight d %}
void assertf(string file = __FILE__, int line = __LINE__, Args...)
            (lazy bool test, lazy Args args)
{
    import std.string : format;
    import core.exception : AssertError;
    version(assert) if (!test)
    {
        throw new AssertError(format(args), file, line);
    }
}
{% endhighlight %}

Note the addition of the `lazy` storage class in the parameter list. This tells
the compiler to not evaluate the arguments until they are used, and since the
arguments will not be used in `-release` builds, they will not be evluated,
solving the final issue with our formatted assert.

That's it! Remember, `version(assert)` won't work until DMD 2.061 (or unless
you get HEAD from git). Until then, you could either just leave it in all builds,
or use `debug` to enable it only in `-debug` builds.

[eval]: http://en.wikipedia.org/wiki/Evaluation_strategy
[lazy]: http://dlang.org/lazy-evaluation.html