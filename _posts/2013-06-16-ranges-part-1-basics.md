---
layout: post
title: Ranges Part 1 - Basics
tags:
- d
- dlang
- ranges
- tutorial
---
Iteration is one of the most common activities in programming. Few programming
tasks can be accomplished without looping or recursing over some set of values,
whether it be a stream of bytes from a file, elements of an array, rows in a
database, or nodes in an implicitly generated graph. Programs need to iterate,
and programming languages need to provide idioms for iteration.

In D, iteration is achieved through the use of _ranges_. Broadly speaking, a
range defines a sequence of values. There are many styles of ranges.
For example, some ranges lazily generate their values instead of iterating
over data; some ranges are infinite; some ranges can be iterated from both
ends; and some ranges allow you to jump around to any index (like arrays). For
the most part, the details of how a particular range operates is up to the
range implementor.

So how do you implement a range? At the most basic level, a range is just a
type with three member functions: `front`, `empty`, and `popFront`.

- `front` returns the value from the front of the range.
- `empty` returns true if the range has been exhausted.
- `popFront` removes the first value from the range.

As a simple example, here is a range that counts from 0 to 'n'.

{% highlight d %}
struct Iota
{
    private int m_current = 0;
    private int m_target;

    this(int n) { m_target = n; }
    @property int front() { return m_current; }
    @property bool empty() { return m_current == m_target; }
    void popFront() { m_current++; }
}
{% endhighlight %}

(The name [`Iota`][iota] is a Greek letter. In a tradition started by the
programming language [APL][apl], it is used to represent consecutive integers
[in][i1] [several][i2] [languages][i3])

Notice that there is no need to inherit from any sort of `IRange` interface.
There's nothing magic about ranges, they are just simple types with those
functions defined.

To iterate over a range, you can manually call those functions using a `for`
loop, or use the built-in `foreach` loop in D, which knows about ranges.

{% highlight d %}
void main()
{
    import std.stdio;
    foreach (x; Iota(10))
        writeln(x);
}
{% endhighlight %}

As expected, this will print out the numbers 0 through 9, one per line. Of
course, `writeln` knows about ranges too, so `writeln(Iota(10))` works as
well, and will print out `[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]`, formatted like an
array.

Writing functions that work with ranges is not difficult. Suppose we want to
write a function to count the number of elements in a range. This can be
achieved by accepting the range as a _template parameter_ and iterating the
range using `popFront()` explicitly, like so:

{% highlight d %}
size_t count(Range)(Range r)
{
    size_t n = 0;
    while (!r.empty)
    {
        ++n;
        r.popFront();
    }
    return n;
}

unittest
{
    assert(count(Iota(10)) == 10);
}
{% endhighlight %}

By using templates, we can create range types that are templated on other
range types. For example, consider a _skip_ range that skips every second
element in another range. You could define it like this:

{% highlight d %}
struct Skip(Range)
{
    private Range m_range;

    this(Range r) { m_range = r; }

    // front and empty just forward to the sub-range.
    @property auto ref front() { return m_range.front; }
    @property bool empty() { return m_range.empty; }

    // popFront also forwards to the sub-range, but pops off two
    // elements at a time, instead of one.
    void popFront()
    {
        m_range.popFront();
        if (!m_range.empty)
            m_range.popFront();
    }
}
{% endhighlight %}

We can then use `Skip` in conjunction with `Iota` to skip through consecutive
integers.

{% highlight d %}
void main()
{
    import std.stdio;
    writeln(Skip!Iota(Iota(10))); // [0, 2, 4, 6, 8]
}
{% endhighlight %}

Notice that we have to redundantly specify `Iota` as a template parameter to
`Skip`. This is because D [doesn't currently support][dip] type inference for
template type constructors. A common workaround is to create a module-level
function that wraps the constructor:

{% highlight d %}
auto skip(Range)(Range r) { return Skip!Range(r); }
auto iota(int n) { return Iota(n); }
{% endhighlight d %}

We can then use these like so:

{% highlight d %}
writeln(iota(10).skip());
{% endhighlight %}

The above code also makes use of another D feature: uniform function call
syntax, which basically means `f(x)` can be written as `x.f()` as if
`f` were a member function of `x`. If you come from the C# world, you can
kind of think of it as every free function being an [Extension Method][ext].
We could have also written `10.iota().skip()` or just plain `skip(iota(10))`.
There's no difference, so choose whatever you think is most readable.

The ability to combine arbitrary ranges is perhaps their most powerful feature.
There's no reason we need to stop at combining just two ranges:

{% highlight d %}
writeln(iota(10)); // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
writeln(iota(10).skip()); // [0, 2, 4, 6, 8];
writeln(iota(10).skip().skip()); // [0, 4, 8];
writeln(iota(10).skip().skip().skip()); // [0, 8];
{% endhighlight %}

It shouldn't be hard to imagine the kind of flexibility and expressiveness that
can be achieved once you have a large vocabulary of ranges at your disposal.

In Part 2 we'll look at some of the different categories of ranges.

[iota]: http://en.wikipedia.org/wiki/Iota
[apl]: http://en.wikipedia.org/wiki/APL_(programming_language)
[i1]: http://en.cppreference.com/w/cpp/algorithm/iota
[i2]: http://dlang.org/phobos/std_range.html#iota
[i3]: http://golang.org/ref/spec#Iota
[dip]: http://wiki.dlang.org/DIP40
[ext]: http://msdn.microsoft.com/en-us/library/vstudio/bb383977.aspx