---
layout: post
title: Small Object Overhead
tags:
- programming
- performance
- design
- optimisation
---
One problem that seems to come up quite a lot is the issue of per-object
overhead when you have lots of small objects. For example, consider a
small class to represent a bullet in a side-scrolling shooter:

{% highlight d %}
struct Bullet
{
    void update(float delta);

    Vec2 m_position;
    Vec2 m_velocity;
}
{% endhighlight %}

That's how it starts at least. What happens when the bullet hits something?
It needs to inform the scoring system that you've scored some points. You'll
also probably want to play some sound effects, perhaps control a sprite; maybe
some bullets increase the health of the player when they hit things. Before
long, your simple `Bullet` class looks more like this:

{% highlight d %}
struct Bullet
{
    void update(float delta);

    ScoringSystem m_scoringSystem;
    AudioSystem m_audioSystem;
    SpriteSystem m_spriteSystem;
    Player m_player;
    Vec2 m_position;
    Vec2 m_velocity;
}
{% endhighlight %}

One solution is to bunch up all of those systems into a separate object.

{% highlight d %}
struct Bullet
{
    void update(float delta);

    Systems m_systems;
    Vec2 m_position;
    Vec2 m_velocity;
}
{% endhighlight %}

This is certainly better for the size of your object (one word overhead vs. four),
but it still smells a bit funky.

A better solution, in my opinion, is to take responsibility away from the `Bullet`.

{% highlight d %}
class BulletSystem
{
    void update(float delta)
    {
        foreach (bullet; m_bullets)
        {
            bullet.m_position += bullet.m_velocity;
            ...
        }
    }

    ScoringSystem m_scoringSystem;
    AudioSystem m_audioSystem;
    SpriteSystem m_spriteSystem;
    Player m_player;
    Bullet[] m_bullets;
}

struct Bullet
{
    Vec2 m_position;
    Vec2 m_velocity;
}
{% endhighlight %}

Now, the `Bullet` is nothing more than the data associated with the bullet itself.
All the logic and responsibility is pushed up into a less granular `BulletSystem`.
This has a few advantages:

1. No unnecessary overhead per bullet.
2. Only one function call to update N bullets, rather than N function calls.
3. Opens up opportunities to parallelize updates.
4. Has much better data flow.

When information and logic are too granular, you increase overhead and lose control,
which means losing opportunities to make high-level optimisations. In this case,
the whole seems to be greater than the sum of the parts.