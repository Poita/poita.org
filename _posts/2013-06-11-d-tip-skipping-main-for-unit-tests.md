---
layout: post
title: D Tip - Skipping main() for Unit Tests
tags:
- d
- dlang
- tip
- unit test
- unittest
- main
---
When you compile a D program with unit tests enabled (using the `-unittest`
flag), the generated executable will first run the unit tests then continue
with executing `main()`.

Sometimes, you just want the unit tests to run in isolation.

A simple way to skip execution of `main()` for unit test builds is to use
the `unittest` version identifier:

{% highlight d %}
unittest
{
    assert(1 + 1 == 2, "Something horrible has gone wrong.");
}

void main()
{
    version(unittest)
        return;

    // This won't be run
}
{% endhighlight %}

Code inside the `version(unittest)` block will only be run when the code is
compiled with `-unittest`, so the code above will exit `main()` immediately
in unit test builds.

There are numerous pre-defined version identifiers, which you can find at
[http://dlang.org/version.html][1].
Another handy one is `version(assert)`, which is only compiled in when asserts
are enabled.

[1]: http://dlang.org/version.html