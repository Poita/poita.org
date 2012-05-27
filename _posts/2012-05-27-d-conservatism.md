---
layout: post
title: D's Conservatism
tags:
- programming
- languages
- conservatism
- d
---
The [D Programming Language][1] is still my current favourite programming
language, but I'm really starting to dislike its conservatism in several
areas. It's great that D tends to take the safe approach to many problems,
but often it is at the expense of expressiveness, and to make matters worse,
there's often no workaround.

One example of this conservatism is with static constructors and destructors.
Static constructors are per-module functions that are run on thread
initialisation and static destructors are run on thread shutdown. The purpose
of these is to initialise and deinitialise global data.

The problem with static constructors and destructors is that you cannot use
them with cyclic imports. For example:

{% highlight d %}
module A;
import B;

int x;
static this()
{
    x = 1;
}
{% endhighlight %}

{% highlight d %}
module B;
import A;

int y;
static this()
{
    y = 2;
}
{% endhighlight %}

This will compile fine, but if you try to run it, you will get an error:

    Cycle detected between modules with ctors/dtors:
    A -> B -> A
    object.Exception@src/rt/minfo.d(331): Aborting!

The reason this is disallowed is because your static constructor could assume
that any imported module's static constructor has already been run. If there
are cycles in the imports then there is no order to run the static constructors
that would allow this assumption, so D disallows it altogether.

The problem is that in this case, there is no cyclic dependency -- it doesn't
matter what order the constructors are run, but D disallows it anyway. D is
being conservative against my will and now I'll have to refactor my code just
to please the compiler.

D's type system also exhibits similar conservatism. `const`/`immutable`,
`shared`, and `pure` all provide their guarantees by conservatively disallowing
anything that could possibly break those guarantees, regardless if they
actually do.

For example, it's not possible to create a pure function that uses a global
store to do caching because pure functions aren't allowed to read or modify
global state.

With `const` and `immutable`, you are forced to use physical constness, which
is a subset of logical constness. This means that you can't do things like
lazy initialisation of data members within a `const` "getter" function, or cache
the result of a `const` function.

D disallows these for a good reason: multithreaded code requires physical
constness in order to avoid data races. The problem is that now interfaces
will be requiring `const` functions even when there is no intention for them
to be multithreaded. For physical constness to work, interfaces should only
require it when concurrency is intended.

These issues have all been raised on the forums, but it all seems to be
ignored. Fortunately it's fairly easy to avoid these issues by not using the
type constructors at all, and avoiding much of the standard library. Hopefully,
useful functions like `std.algorithm.sort` won't start requiring pure functions
as predicates.

[1]: http://dlang.org