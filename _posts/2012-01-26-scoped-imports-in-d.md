---
layout: post
title: Scoped Imports in D
tags:
- d
- programming
- tip
---
One of the nice things about the [D programming language][1] is that it
has very convenient syntax for conditional compilation. For example,
suppose you want to print out some useful info in debug builds. You just
use:

{% highlight d %}
debug writeln("Value of x is ", x);
{% endhighlight %}

One problem that I constantly run into in cases like this is that I
haven't `import`ed `std.stdio`, so I have to go right up to the top of
the file, add `debug import std.stdio;`, back down again, and continue.
*Sigh*.

Not so fast! D has scoped imports, so you can put the `import
std.stdio;` right where you need it.

{% highlight d %}
void main(string[] args)
{
  debug import std.stdio;
  debug writeln(args);
  // ...
}
{% endhighlight %}

It's the little things like this that make D so pleasant to use.

[1]: http://dlang.org
