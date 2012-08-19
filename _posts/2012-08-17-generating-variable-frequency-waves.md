---
layout: post
title: Generating Variable Frequency Waves
tags:
- dsp
- waves
- frequency
- programming
- math
- phase
---
I've been playing around with some simple digital signal processing recently --
generating PCM streams and playing them back with OpenAL. The first thing I tried
was a simple sine wave. Coding this up is easy enough:

{% highlight d %}
uint sampleRate = 16000;
uint numSamples = sampleRate * 10;
double[] samples = new double[numSamples];
double freq = 440;
foreach (i; 0..numSamples)
{
    double t = cast(double) i / sampleRate;
    samples[i] = sin(2 * PI * freq * t);
}
{% endhighlight %}

How about a wave whose frequency oscillates up and down, like a siren? Just
oscillate the `freq`, right?

{% highlight d %}
freq = 440 + 100 * sin(2 * PI * t);
samples[i] = sin(2 * PI * freq * t);
{% endhighlight %}

Looks good, but if you play that back, you'll notice that something's wrong.
The frequency goes up and down, but much more than expected, and at unexpected
rates.

What's wrong? The problem is the in the `2 * PI * freq * t`.

It's easy to see why this is wrong by considering a different frequency function.
Suppose we had instead:

{% highlight d %}
freq = 440 / t;
samples[i] = sin(2 * PI * freq * t);
{% endhighlight %}

If you substitute `freq` into the `sin` argument, the `t`s will cancel out,
leaving the argument -- and whole expression -- as a constant, which is
not going to produce a wave of any form.

The problem is that the argument to `sin` should be the current phase. When
the frequency is constant (like in the first code snippet) the current phase
is given by `freq * t`, but in general the phase is:

`\[ \int_0^t \! f(t) \, \mathrm{d} t. \]`

where `\( f(t) \)` is the frequency as a function of time.

When you are generating a discrete-time waves, the integral becomes a sum.
We can realise this in code by using what's called a [phase accumulator][1].

{% highlight d %}
double phase = 0;
double dt = 1.0 / sampleRate; // time between samples
foreach (i; 0..numSamples)
{
    samples[i] = sin(2 * PI * phase); // use accumulated phase

    double t = cast(double) i / sampleRate;
    freq = 440 + 100 * sin(2 * PI * t);
    phase += freq * dt; // accumulate phase
}
{% endhighlight %}

As you can see, the phase accumulator simply adds up the frequency multiplied
by the delta time. You might recognise this as an [Euler integrator][2].

Using this method, you can vary the frequency however you like, even in
discontinuous ways, and still get a nice smooth sine wave.

[1]: http://en.wikipedia.org/wiki/Numerically_controlled_oscillator#Phase_accumulator
[2]: http://en.wikipedia.org/wiki/Euler_integration
