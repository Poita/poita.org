---
layout: post
title: API Performance
tags:
- programming
- api
- performance
- process
---
One thing that I've never given much thought about until recently is the
performance of APIs. I don't mean things like avoiding `virtual`
function calls in your interfaces. I mean designing an API so that it
promotes usage that is optimal for the hardware you are running on.

A good example of this was explored in Noel Llopis' post [*Data-Oriented
Design Now And In The Future*][1]. Imagine you are designing the API for
a physics system. One key operation of a physics system is to perform
ray casts. The obvious API for this would be:

{% highlight d %}
RayHitInfo rayCast(PhysicsWorld, Ray);
{% endhighlight %}

Many physics systems use an API like this. For example, it's very
similar to [the API][2] provided by Unity3D.

Most game engines also have some sort of game object system, whose main
task usually involves looping through each game object, calling an update
function. Many of those update functions will need to perform ray casts;
perhaps something like this:

{% highlight d %}
void update()
{
  // ...
  if (rayCast(world, forwardRay))
    doSomething();
  // ...
}
{% endhighlight %}

If you have many objects doing this then you are going to run into
performance problems. The ray cast is going to touch lots of data (and
code) to perform the query, causing multiple cache misses as it
traverses through space partitioning data structures, mesh data, etc.
Then, once the ray cast is complete, you will continue processing update
functions, likely pushing most or all that ray cast data and code out of the
cache. When the next ray cast comes up, you have to pull it all back in
again.

In addition to the poor memory performance, you also can't parallelize
the ray casts. The call is blocking, so no other object updates can
happen until the ray cast completes, meaning that no other ray casts are
available for parallelization.

From a performance point of view, the ideal use case would be to
1. Gather all the queries up front from each game object.
2. Batch process those queries.
3. Feed the results back to game objects to continue updating.

The key thing to note here is that **this ideal is impossible to achieve
with the current API**. In order to make this change, you will have to
go through and re-shuffle the logic for each raycasting update function.

If you want a performant API then it is something that should be
considered beforehand, as it may not be easy to fix later on.
In general, APIs that encourage asynchronicity and batched execution
tend to have higher potential for good performance than synchronous,
one-at-a-time functions.

[1]: http://gamesfromwithin.com/data-oriented-design-now-and-in-the-future
[2]: http://unity3d.com/support/documentation/ScriptReference/Physics.Raycast.html
