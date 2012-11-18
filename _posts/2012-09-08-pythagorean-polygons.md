---
layout: post
title: Pythagorean Polygons - Part 1
tags:
- d
- project euler
- programming
- math
- pythagorean polygons
- dynamic programming
---
It's been a while since I solved any coding problems, so I've decided to
re-solve a [Project Euler][euler] problem that I looked at years ago:
[Pythagorean Polygons][p292]. Back then I used C++, but this time I'll use
the D programming language :-)

In summary, the problem is to find the number of distinct convex polygons with
vertices on integer coordinates, integer length sides, and a perimeter
less than or equal to 120. The name of the problem comes from the fact
that the edges are the hypotenuses of [Pythagorean triples][triples].

The brute force method is quite simple:

1. Generate all the Pythagorean triples (x, y, d) with d < 120.
2. Do a depth first search on the integer lattice from (0, 0) to (0, 0),
using the triples as your edges, ensuring that you always turn anti-clockwise
(or always clockwise).

Generating the Pythagorean triples is easy enough:

{% highlight d %}
enum N = 120;    // Maximum perimeter
enum M = N / 2;  // Maximum edge length

struct V { int x, y, d; }
V[] vs;

foreach (dx; -M..M+1)
    foreach (dy; -M..M+1)
        foreach (dd; 1..M+1)
            if (dx * dx + dy * dy == dd * dd)
                vs ~= V(dx, dy, dd);
{% endhighlight %}

It will also help to have them in anti-clockwise order.

{% highlight d %}
double arg(V v)
{
    import std.math;
    return atan2(cast(real)v.y, cast(real)v.x);
}

import std.algorithm;
sort!((a, b) => arg(a) < arg(b))(vs);
{% endhighlight %}

Our triples are likely to contain several colinear edges. So we'll need a way
of avoiding those:

{% highlight d %}
bool colinear(V a, V b)
{
    // a and b are colinear if a.x / a.y == b.x / b.y
    // We can avoid the division by multiplying out.
    return a.x * b.y == a.y * b.x;
}

int next(int i)
{
    // Find next, non-colinear edge
    int j = i + 1;
    while (j < vs.length && colinear(vs[i], vs[j]))
        ++j;
    return j;
}
{% endhighlight %}

Now all that's left is to do the depth-first search. I'm just going to use
simple recursion to implement it, just to save code.

{% highlight d %}
// Returns number of paths from (x, y) to (0, 0) using less
// than d perimeter length, and only using edges with
// index >= i (to ensure convexity).
long dfs(int x, int y, int d, int i)
{
    if (d > N) return 0; // Perimeter too long
    if (x == 0 && y == 0 && d > 0) return 1;

    // Iterate edges, add them on, and recurse.
    long total = 0;
    foreach (int k; i..cast(int)vs.length)
        if (!(d == vs[k].d && x == -vs[k].x && y == -vs[k].y))
            total += dfs(x + vs[k].x, y + vs[k].y, d + vs[k].d, next(k));
    return total;
}
{% endhighlight %}

The `if`-condition inside the loop is to protect against degenerate polygons with
two edges e.g. (0, 0), (60, 0), (0, 0), and the use of `next` ensures that we
don't use two colinear edges in a row, as this is disallowed in the problem.

Finally, we need to call it with the starting position:

{% highlight d %}
import std.stdio;
writeln(dfs(0, 0, 0, 0));
{% endhighlight %}

If we change N to 30, we can test it using the given solutions in the problem.

{% highlight c %}
% dmd -inline -O -release p292
% time ./p292
3655
{% endhighlight %}

Great. That matches with the solution and runs in under 2 seconds on my laptop.
Unforunately, trying it with N = 60 took just shy of 50 **minutes**.

The problem with this is that the branching factor of the search
is equal to the number of triples (hundreds) and the depth exponent is large 
enough to make the running time impractical for larger N. We need a way to speed
it up.

Let's look at the signature of `dfs`.

{% highlight d %}
long dfs(int x, int y, int d, int i)
{% endhighlight %}

What's the domain of each input?

`x` and `y` will be from -60 to 60.  
`d` will be between 0 and 120.  
`i` will be between 0 and the number of triples (448).  

That gives us a maximum of 121 x 121 x 121 x 448 possible inputs. Since `dfs`
is a pure function, that means we can memoize the results and re-use them
for subsequent recursive calls. It's a large number, but now the running time
is polynomial rather than exponential, so it will scale better.

Fortunately, D's standard library, Phobos, provides a [`memoize`][memo] template
just for this purpose. With a couple of quick modifications to `dfs`, we can
memoize it.

{% highlight d %}
long dfs(int x, int y, int d, int i)
{
    import std.functional;
    alias memoize!dfs mdfs;

    // ...
            total += mdfs(x + vs[k].x, y + vs[k].y, d + vs[k].d, next(k));
    // ...
}
{% endhighlight %}

We just introduce an `alias` for the memoized version of the function, and
change the recursive call to use that version instead. Internally, `memoize`
just maintains a hashtable of input tuples to outputs, so it's quite fast.

With these changes, the N = 60 version is reduced from 50 minutes to 42 seconds.
Unfortunately, N = 120 is still just out of reach. It will likely run in under
an hour if you have enough memory, but we'd like it to run a lot faster than
that.

In the next post, I'll show how we can get N = 120 running in sub-second times.

[euler]: http://projecteuler.net
[p292]: http://projecteuler.net/problem=292
[triples]: http://en.wikipedia.org/wiki/Pythagorean_triple
[memo]: http://dlang.org/phobos/std_functional.html#memoize