---
layout: post
title: DustMite - Code Minimizer
tags:
- dustmite
- d
- dlang
- programming
- bugs
---
Yesterday I discovered [DustMite][1], a source code minimizer tool for D.

What's a source code minimizer? It's a program that takes some source
code, and automatically removes lines, or even whole files from the source
while maintaining some invariant (e.g. it still builds, or still gives
a particular output). The usual use case is to produce a minimal reproduction
test case for a bug, which is exactly what I used it for.

I had a problem in a medium-sized D project where anytime I had some sort
of compile-time error, such as a typo in a variable name, DMD (the D compiler),
would spew out pages and pages of false errors, triggered by some other module
in my codebase. The errors were completely irrelevant. This was
quite annoying because it made it difficult to see the actual error. So, like
a good programmer, I decided to file a bug report.

Step one in filing a good bug report is to create a minimal test case. I tried
to do this manually by pulling out what seemed to be the relevant parts of the
code that would reproduce the error, but I couldn't get it to happen. I could
easily and consistently reproduce the bug in the full codebase, but the project
is about 10,000 lines of code, so minimizing it manually from there would be very
time consuming.

I'd heard about DustMite on the [forums][2], so I thought I'd give it a go.

Installing was easy enough.

{% highlight sh %}
% git clone https://github.com/CyberShadow/DustMite.git
% cd DustMite
% dmd dustmite.d dsplit.d
% ln -s ~/DustMite/dustmite ~/bin
{% endhighlight %}

Next step was to prepare the project for minification. All you need to do is
make a full copy of the codebase, and remove any unnecessary files (e.g. object
files, data files -- anything not needed for reproduction).

Next you need to devise a test command. This is a command that should return 0
if the bug is still present, and non-zero otherwise. For example, suppose we
had this (trivial) program:

{% highlight d %}
import std.stdio;
import std.string;

string world() { return "world!"; }

void main()
{
	write(hello());
	write(", ");
	write(world());
	write("\n");
}
{% endhighlight %}

I've got an obvious bug here in that `hello()` is undefined. Trying to compile
this gives `Error: undefined identifier hello`. If we wanted to minimize this
then we need a command, which, when run, will return 0.

Compiling and `grep`ing for the error would be perfect for this.

{% highlight sh %}
% dmd test.d 2>&1 | grep -q "undefined identifier hello"
% echo $?
0
{% endhighlight %}

This runs the compiler (`dmd test.d`), redirects the error messages to stdout
(`2>&1`) then pipes the result (`|`) to `grep`, which searches for that string
without producing output (`-q` = quiet). `grep` returns 0 if it finds anything,
and 1 otherwise.

The DustMite wiki provides a list of [useful test scripts][3].

Finally, we just run DustMite with that test command.

{% highlight sh %}
% dustmite testdir 'dmd test.d 2>&1 | grep -q "undefined identifier
hello"'
None => Yes
############### ITERATION 0 ################
[  0.0%] Remove [] => No
[  1.6%] Remove [0] => No (cached)
[  3.3%] Remove [01] => No
...
[ 82.4%] Remove [000001] => No (cached)
[ 88.2%] Remove [000000] => No (cached)
[ 94.1%] Remove [0000000] => No (cached)
Done in 36 tests and 10 secs and 165 ms; reduced version is in
testdir.reduced
{% endhighlight %}

And checking `testdir.reduced/test.d` confirms that the source has been reduced.

{% highlight sh %}
% cd testdir.reduced
% cat test.d
void main()
{
hello;
}
{% endhighlight %}

For my project, DustMite took 8 minutes, and reduced roughly 10,000 lines of
code down to about 10, and perfectly reproduced the issue. The problem I was
having trying to reproduce manually was that the repro required that you
pass the files in a specific order to the compiler, and it also required an
extra file that seemingly has nothing to do with the issue. I don't think I
would have ever minimized it without DustMite.

[1]: https://github.com/CyberShadow/DustMite
[2]: http://http://forum.dlang.org/
[3]: https://github.com/CyberShadow/DustMite/wiki/Useful-test-scripts