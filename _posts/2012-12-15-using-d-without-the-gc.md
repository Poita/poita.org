---
layout: post
title: Using D Without The GC
tags:
- d
- dlang
- gc
- performance
- programming
---
One of the major selling points of the [D Programming Language][1] is its
"native efficiency". I use the scare quotes because while D compilers do
compile down to native machine code, your program will only run efficiently
if you [code with performance in mind][2]!

In D, the easiest way to degrade performance (besides poor algorithms) is
overuse of automatic memory managment. No matter what language you use,
memory allocation is non-trivial, and has a non-trivial cost to go with it.
With D's automatic memory management, you not only incur the cost of
allocation, but also the cost of automatic deallocation, i.e. garbage collection.
If you want high throughput and low latency then you really need to avoid
memory allocation as much as possible.

The most important thing to understand about the GC is that it can only run
when you try to allocate. Unfortunately D doesn't make it obvious when you
are allocating memory, so I've listed the most common ways here.

## Things That Allocate in D
Before I start, I should make it clear that these are things that *could*
allocate in D. A good optimiser may be able to remove some of the allocations.
When in doubt, test.

### new
Any time you use the `new` keyword, you are allocating memory. Avoid creating
class instances frequently, and use structs where possible.

### Array literals
In most cases, array literals will allocate.

{% highlight d %}
immutable int[3] a = [1, 2, 3];  // Won't allocate
immutable int[] b = [1, 2, 3];   // Will allocate once
int[] c = [1, 2, 3];             // Will allocate
int[3] d = [1, 2, 3];            // Will allocate
foo([1, 2, 3]);                  // Will allocate
{% endhighlight %}

The fact that the initialisation of `d` allocates may be surprising. It
doesn't need to, but DMD currently doesn't do the optimisation to
remove the allocation. In the meantime, you can do this:

{% highlight d %}
immutable int[3] a = [1, 2, 3];  // Won't allocate
int[3] b = a;                    // Won't allocate
{% endhighlight %}

Calling `dup` or `idup` on an array will also lead to an allocation.

### Array concatenation
Array concatenation won't _always_ result in an allocation, as arrays are
sometimes over-allocated, but if you want to avoid allocations then it is best
to avoid array concatenation. Modifying the `.length` property of an array
could also cause allocation.

### Associative arrays
Creating, adding, or removing elements from an associative array could cause
an allocation. Lookups are safe.

### Closures
A [closure][3] is a function reference that needs extra context beyond the stack.
An example should illustrate this:

{% highlight d %}
int delegate() foo(int x) { return () => x; }
int delegate() bar(int x) { return () => 1; }
{% endhighlight %}

Here, `foo` returns a closure because the returned function relies on the value
of `x`, which will no longer be on the stack after `foo` returns. `bar`,
on the other hand, does not return a closure, because the function `() => 1` has
no dependency on `x`.

In D, creating a closure allocates memory, i.e. every time you call `foo` you
will allocate memory.

## Detecting Allocations
The easiest way to detect unexpected allocations is to add some logging to
the [D runtime][4]. Clone that project, and open up `src/gc/gc.d`. In there,
there's some functions with names like `gc_malloc`, `gc_calloc`, etc. Just add
some calls to `printf` there, and [build][5].

## Style
When you try to avoid using the GC, you might find that your usual style
of programming doesn't work very well. If you are used to just building
up arrays using concatenation, joining strings together, and creating lots
of temporary objects then you're going to have a hard time.

In general, to avoid these dynamic allocations, you need to make your
program more static. Static arrays in particular are your friend. You'll have
to build hard limits into your code, and just "allocate" within those limits.

Using strings can be quite tricky when you want to avoid allocations. `string`
in D is an alias for `immutable(char)[]`, which is fine when you know what
your string needs to contain beforehand, but if you want to create a string
based on runtime data then you'll need to build it up inside a `char[N]`,
which has no legal conversion to `immutable(char)[]`.

To get around the string problem, you can use [`assumeUnique`][6]. This is 
a simple function that does nothing more than cast a mutable array to an
immutable array. The caveat is that the cast is only safe if the reference
is truly unique. Once you have built up your string in the mutable array, and
casted using `assumeUnique`, you can't safely use the mutable array again.
This shouldn't be a problem, but it's something to be aware of.

{% highlight d %}
char[16] buf;
snprintf(buf.ptr, buf.length, "1 + 1 = %d", 1 + 1);
string str = assumeUnique(buf[]);
{% endhighlight %}

## Conclusion
While D was designed with GC use in mind, the language also provides all
the necessary abstraction facilities to build usable "GC free" libraries
(templates, aliases, delegates, static arrays). It can take a while to get
used to not using the GC, but I certainly don't feel hindered by its absence.


[1]: http://dlang.org
[2]: /2012/01/25/api-performance.html
[3]: http://en.wikipedia.org/wiki/Closure_(computer_science)
[4]: https://github.com/D-Programming-Language/druntime
[5]: http://wiki.dlang.org/Building_DMD
[6]: http://dlang.org/phobos/std_exception.html#assumeUnique