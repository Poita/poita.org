---
layout: post
title: Range-Based Graph Search in D
tags:
- d
- dlang
- ranges
- graph
- algorithms
---
I've just made my first commit of a range-based [graph library][1] for D. At the
moment it only contains a few basic search algorithms (DFS, BFS, Dijsktra, and
A\*), but I wanted to write a bit about the design of the library, and how you
can use it.

As a bit of a teaser, the snippet below shows how you can use the library to
solve those word change puzzles (you know, the ones when you have to get from
one word to another by only changing one letter at a time?)

{% highlight d %}
string[] words = File("/usr/share/dict/words")
                 .byLine
                 .filter!(w => w.length == 5)
                 .map!"a.idup"
                 .array();
enum string start = "hello";
enum string end = "world";
static d(string a, string b) { return zip(a, b).count!"a[0] != a[1]"; }
auto graph = implicitGraph!(string, u => words.filter!(v => d(u, v) == 1));
graph.aStarSearch!(v => d(v, end))(start).find(end).path.writeln();
{% endhighlight %}

This promptly produces the desired result:

{% highlight d %}
["hello", "hollo", "holly", "molly", "mouly", "mould", "would", "world"]
{% endhighlight %}

The last two lines are the interesting part. The first of those lines defines
our graph as an *implicit graph* of words connected by one-letter changes. The
second line performs the [A\* search][4] and writes the resultant path to `stdout`.

On the second-last line, `graph` is defined as an `implicitGraph`.
An [implicit graph][2] is a graph that is represented by functions rather than
in-memory data structures. Instead of having an array of adjacency lists, we
just define a function that returns a range of adjacent words (that's the
second parameter to `implicitGraph`). This representation saves us the tedium
of generating the graph beforehand, and saves your RAM from having to store it
unnecessarily. It also makes for succinct graph definitions!

The last line is where the algorithm is called. Unlike most other graph
libraries, `stdex.graph` implements graph searches as ranges -- iterations
of the graph vertices. `aStarSearch` returns a range of vertices in the
order they would be visited by the A\* search algorithm (parameterized by the
A\* heuristic). By implementing the search as a range, the user can take
advantage of [Phobos][3]' multitude of range algorithms.

For instance, suppose you want to count the number of nodes searched by the
algorithm until it reaches the target node. For this task, you can just use
`countUntil` from [`std.algorithm`][5].

{% highlight d %}
auto search = graph.aStarSearch!(v => d(v, end))(start);
writefln("%d nodes visited", search.countUntil(end));
{% endhighlight %}

For tuning and debugging, it might be useful to print out the nodes visited
by the algorithm as it runs.

{% highlight d %}
foreach (node; search.until(end))
    writeln(node);
{% endhighlight %}

When no path is available, graph search algorithms typically end up searching
the entire vertex space. It is common to cut-off the search after a certain
threshold of nodes have been searched. This can be accomplished by `take`ing
only so many nodes from the search.

{% highlight d %}
auto result = search.take(50).find(end);
if (result.empty)
    writeln("Not found within 50 nodes");
{% endhighlight %}

Similarly, you could have the search continue until 10 seconds have elapsed.

{% highlight d %}
alias now = TickDuration.currSystemTick;
auto t = now;
auto result = search.until!(_ => (now - t).seconds > 10).find(end);
if (result.empty)
    writeln("Not found in 10 seconds");
{% endhighlight %}

The beauty of all this is that none of these features are part of the graph
library -- they come for free as a side-effect of being a range.

One unsolved part of the design for the graph searches is how to handle
visitation. The Boost Graph Library solves this by having users implement
a visitor object, which has to define a suite of callback methods, one for
each event (vertex discovered, examine edge, etc.) This is certainly mimicable
in D, but it may be more effective to model visitation with output ranges,
which would again allow composition with existing Phobos constructs. This is
what I will be investigating next.

[1]: https://github.com/Poita/stdex/blob/master/graph.d
[2]: http://en.wikipedia.org/wiki/Implicit_graph
[3]: http://dlang.org/phobos/
[4]: http://en.wikipedia.org/wiki/A*
[5]: http://dlang.org/phobos/std_algorithm.html