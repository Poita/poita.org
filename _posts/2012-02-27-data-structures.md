---
layout: post
title: Data Structures
tags:
- programming
---
Something I've realised recently is that it's usually a bad idea to use data
structures to structure your data... yeah.

That probably doesn't make much sense. What I mean is that you shouldn't use
data structures to *organise* high-level information in your programs. Data
structures exist to make operations and algorithms faster, and should have
nothing to do with how your "objects" are organised logically.

As an example, consider a game that has several worlds, and within each world
there are several stages. Each stage has a bunch of stars you have to collect,
and we want to know how many stars there are and how many you've collected.

How might you structure this in your program? Until recently, I would have
used something like this:

{% highlight d %}
class Game
{
    World[] worlds;
}

class World
{
    Stage[] stages;
}

class Stage
{
    Star[] stars;
}

class Star
{
    bool collected;
}
{% endhighlight %}

This seems reasonable. Let's try to use it:

{% highlight d %}
int totalStars(Game game)
{
    int count = 0;
    foreach (world; game.worlds)
        foreach (stage; world.stages)
            count += stage.stars.length;
    return count;
}

int starsCollected(Game game)
{
    int count = 0;
    foreach (world; game.worlds)
        foreach (stage; world.stages)
            foreach (star; stage.stars)
                if (star.collected)
                    ++count;
    return count;
}
{% endhighlight %}

I don't know about you, but I'm not impressed. It's not too
difficult to write these functions, but I can't help but thinking, "what am I
gaining from structuring the code like this?"

If we step back and look at the problem that needs to be solved, surely the
following would be a better way to structure the data?

{% highlight d %}
class Game
{
    Star[] stars; // ALL the stars
}

class Star
{
    bool collected;
    int stageId;
}
{% endhighlight %}

This way, the queries are straightforward loops over all the stars. If we
want to count only the stars in a particular stage then we can filter using the
star's `stageId`.

In the end, what matters is how you *use* the data, and what algorithms need to
be implemented efficiently on the data. Any relationship between things in the
application domain is of little relevance when choosing data structures.


