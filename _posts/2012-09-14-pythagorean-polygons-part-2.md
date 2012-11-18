---
layout: post
title: Pythagorean Polygons - Part 2
tags:
- d
- project euler
- programming
- math
- pythagorean polygons
- dynamic programming
---
In my [last post][part1] I showed how to solve the Project Euler problem,
Pythagorean Polygons for N = 60 in under a minute. In this post I'll show how
to solve it for the target N = 120, and in under a second.

Our current algorithm uses a brute force depth first search, memoized using
the generic Phobos memoize template. One common way to speed up searches is
to prune branches that have no possibility of reaching the goal. In this problem,
we can prune a search when the distance to the goal is greater than the 
remaining perimeter:

{% highlight d %}
long dfs(int x, int y, int d, int i)
{
    import std.functional;
    alias memoize!dfs mdfs;

    if (x == 0 && y == 0 && d > 0) return 1;

    // Recurse
    long result = 0;
    foreach (int k; i..cast(int)vs.length)
        if (!(d == vs[k].d && x == -vs[k].x && y == -vs[k].y))
        {
            int nx = x + vs[k].x;  // new X
            int ny = y + vs[k].y;  // new Y
            int nd = d + vs[k].d;  // new used perimeter
            int rd = N - nd;       // remaining perimeter
            if (nx * nx + ny * ny <= rd * rd && rd >= 0)
                result += mdfs(nx, ny, nd, next(k));
        }
    return result;
}
{% endhighlight %}

This alone speed up the N = 60 solution to under 2 seconds, and gives us the
solution to N = 120 in just over 2 minutes.

We can do better.

In the N = 120 case, the memoization cache uses up roughly 200MB of memory.
This is much larger than my L2 cache, and since hash maps distribute elements
randomly, we'll be spending a lot of time going out to memory checking the 
memoization cache. Reducing memory usage and improving lookup patterns should
help performance significantly.

One observation we can make is that `dfs(..., i)` always calls
`dfs(..., next(i))`. If we change the algorithm from a top-down memoization
algorithm to a bottom-up dynamic programming algorithm, then we could do the
depth-first search backwards, in layers of different values of `i`. We start
by computing `dfs(..., vs.length-1)` for all `x`, `y`, and `d`. That gives us
everything we need to compute all of `dfs(..., prev(vs.length-1))`.

`prev` is just the opposite function to `next` -- it returns the index `j`,
previous to `i` such that `vs[i]` is not colinear with `vs[j]`.

{% highlight d %}
int prev(int i)
{
    // Find next, non-colinear edge
    int j = i - 1;
    while (j >= 0 && colinear(vs[i], vs[j]))
        --j;
    return j;
}
{% endhighlight %}

Since we aren't going to be caching all of `dfs` anymore, we'll get rid of
the memoization and instead just use a big array as our cache. We'll also need
[two of them][buffering]: one for the previously computed layer of `i`, and one for the next
layer that is being computed.

{% highlight d %}
alias long[N+1][N+1][N+1] DP; // cache
DP[] dp = new DP[2];
dp[0][M][M][0] = 1; // start point
int r = 0, w = 1; // read index, write index
{% endhighlight %}

Here, `dp[r]` stores the previously computed layer. `dp[r][M+x][M+y][d]` represents
the number of paths of length `d` from (0, 0) to (x, y) using the only the triples
from the previous layers. Using `dp[r]` we compute `dp[w]` by using the next layer
down from what `dp[r]` used. To do this, we start by copying all of `dp[r]` into
`dp[w]` and then simply iterate over all `x`, `y`, and `d`, as well as all colinear
edges in the current layer.

The algorithm looks something like this:

{% highlight d %}
for (int i = cast(int)vs.length - 1; i >= 0; )
{
    dp[w] = dp[r];    // copy previous buffer
    auto j = prev(i); // colinear edges from j..i

    foreach (x; -M..M+1)
    foreach (y; -M..M+1)
    foreach (d; 0..N+1)
        if (dp[r][M+x][M+y][d] > 0)
            for (int k = i; k > j; --k)
                if (!(d == vs[k].d && x == -vs[k].x && y == -vs[k].y))
                {
                    int nx = x + vs[k].x;
                    int ny = y + vs[k].y;
                    int nd = d + vs[k].d;
                    int rd = N - nd;
                    if (nx * nx + ny * ny <= rd * rd && rd >= 0)
                        dp[w][M+nx][M+ny][nd] += dp[r][M+x][M+y][d];
                }

    i = j;       // move to next layer
    swap(r, w);  // swap read/write buffers
}
{% endhighlight %}

Once all this has run for each layer, `dp[r]` will contain the final path counts for each
(x, y, d) tuple. All that's left to do is count up the number of paths that
reach back to (0, 0) (`dp[r][M][M][...]`).

{% highlight d %}
alias reduce!("a + b") sum;
long total = sum(dp[r][M][M][1..N+1]);
writeln(total);
{% endhighlight %}

With this improvement, the N = 120 case now runs in just over 1 second, which
is easily fast enough for us to be satisfied, but there's much more that can
be done.

Like many geometrical problems, Pythagorean Polygons has a symmetry to it that
can be exploited for speed. Notice that we generate the Pythagorean triplets
redundantly from all quadrants. If (3, 4) is a triplet, then so is (-3, 4),
(3, -4), (-3, 4), (4, 3), (-4, 3), (4, -3), and (-4, -3). What this also means
is that if there is a path from (0, 0) to (x, y, d) then there are also paths
to (-x, y, d), (x, -y, d) etc. but we compute all of these redundantly!

Taking *full* advantage of the symmetry is tricky, but there's one simple way
that can speed up our algorithm up significantly. Notice that the first half of
triplets travel upwards, and the second half travel downwards (due to the sort).
This means our algorithm calculates all paths down to some point (x, y, d) using
the second half of the triplets, then calculates all paths from there back to
(0, 0) using the second half of the triplets. But we know from the symmetry that
if there is a path to (x, y, d) using the first half then there is a path back
to (0, 0) using the second half. In fact, there are the same number of paths,
so the total unique paths is equal to the product of all combinations where the
total distance is less than N.

To implement this, we only need to make one simple change, and add a combining
step. The change is to modify the main loop to only use the first half of the
triplets, i.e.

{% highlight d %}
for (int i = cast(int)vs.length - 1; i >= 0; )
{% endhighlight %}

becomes

{% highlight d %}
for (int i = cast(int)vs.length / 2 - 1; i >= 0; )
{% endhighlight %}

Next, instead of just summing up the `dp[r][M][M][1..N]` entries, we need to
sum up the path products. To do this, we iterate over all (x, y, d) tuples and
then multiply the cache value with those for (x, y, d2) where d2 <= N - d.

{% highlight d %}
long total = 0;
foreach (x; -M..M+1)
foreach (y; -M..M+1)
foreach (d; 1..N+1)
    if (dp[r][M+x][M+y][d] > 0)
        foreach (d2; 0..N+1-d)
            total += dp[r][M+x][M+y][d] * dp[r][M+x][M+y][d2];
total -= vs.length / 2;
{% endhighlight %}

That last line needs some explanation. Previously our algorithm accounted for
paths with only two edges, but we don't do that anymore, so they will be included
in the totals. What that line of code does is remove those paths from the total.
That value is `vs.length / 2` because every edge (dx, dy) has a corresponding edge
(-dx, -dy) that will be included, so the total of these two-edge paths is equal
to half the triplets.

With this improvement, the N = 120 case gives the solution in 0.36 seconds,
which is more than fast enough for my liking. The other axes of symmetry can
be exploited further, but I'll leave that as an exercise for the reader :-)

[part1]: /2012/09/08/pythagorean-polygons.html
[buffering]: http://en.wikipedia.org/wiki/Multiple_buffering