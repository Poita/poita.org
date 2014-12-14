---
layout: post
title: The Condenser Part 1
tags:
- condenser
- programming
- dlang
- joe armstrong
- duplicates
---
Last night I rewatched Joe Armstrong's excellent [keynote][key] from
this year's Strange Loop conference. He begins the talk with a fairly
standard, but humorous lament about the sorry state of software, then
goes on to relate this to some physical quantities and limits. He finishes
by suggesting some possible directions we can look to improve things.

One of his suggestions is to build something called "The Condenser", which
is a program that takes all programs in the world, and condenses them down
in such a way that all redundancy is removed. He breaks this task down
into two parts:

![The Condenser](/img/the-condenser.png)

He also explains the obvious, and easy way to do Part 1.

![The Condenser Part 1](/img/the-condenser-part-1.png)

At this point I remembered that I hadn't written a lot of D recently, and
although Joe is talking about condensing all files _in the world_, I was
kind of curious how much duplication there was just on my laptop. How many
precious bytes am I wasting?

It turns out that The Condenser Part 1 is very easy to write in D, and even
quite elegant.

{% highlight d %}
import std.algorithm;
import std.digest.sha;
import std.file;
import std.parallelism;
import std.stdio;
import std.typecons;

auto process(DirEntry e) {
  ubyte[4 << 10] buf = void;  // 4kb buffer, uninitialized
  auto sha1 = File(e.name).byChunk(buf[]).digest!SHA1;
  return tuple(e.name, sha1, e.size);
}

void main(string[] args) {
  string[ubyte[20]] hash2file;  // hash table of SHA1 -> filename
  auto files = dirEntries(args[1], SpanMode.depth, false)
               .filter!(e => e.isFile);
  foreach (t; taskPool.map!(process)(files)) {
    if (string* original = t[1] in hash2file) {
      writefln("%s %s duplicates %s", t[2], t[0], *original);
    } else {
      hash2file[t[1]] = t[0];
    }
  }
}
{% endhighlight %}

I'll explain the code a little, starting from `main`. The `files` variable
uses `dirEntries` and `filter` to produce a list of files (the `filter` is
necessary since `dirEntries` also returns directories). `dirEntries` is a
lazy range, so the directories are only iterated as necessary. Similarly,
`filter` creates another lazy range, which removes non-file entries.

The `foreach` loop corresponds to Joe's `do` loop. While I could have just
iterated over `files` directly, I instead iterate over a `taskPool.map`, a
semi-lazy map from David Simcha's [`std.parallelism`][par] that does the
processing in parallel on multiple threads. It is semi-lazy because it
eagerly takes a chunk of elements at a time from the front of the range,
processes them in parallel, then returns them back to the caller to consume.
The task pool size defaults to the number of CPU cores on your machine, but
this can be configured.

The `process` function is where the hashing is done. It takes a `DirEntry`,
allocates an uninitialized (`= void`) 4kb buffer on the stack and uses that
buffer to read the file 4kb at a time and build up the SHA1. Again, `byChunk`
returns a lazy range, and `digest!SHA1` consumes it. Lazy ranges are very
common in idiomatic D, especially in these kinds of stream processing type
applications. It's worth getting familiar with the common ranges in
[`std.algorithm`][alg] and [`std.range`][ran].

I ran the program over my laptop's home directory. Since the duplicate
file sizes are all in the first column, I can just use `awk` to sum them
up and find the total waste:

{% highlight sh %}
$ sudo ./the-condenser . | awk '{s+=$1} END {print s}'
2347353793
{% endhighlight %}

That's 2.18Gb of duplicate files, which is around 10% of my home dir.

In some cases the duplicates are just files that I have absentmindedly
downloaded twice. In other cases it's duplicate music that I have both in
iTunes and Google Play. For some reason, the game Left 4 Dead 2 seems to have
a lot of duplicate audio files.

So, in theory, I could save 10% by deduplicating all these files, but it's
not a massive amount of waste. It doesn't seem worth the effort trying to
fix this. Perhaps it would be nice for the OS or file system to solve this
automatically, but the implementation would be tricky, and still maybe not
worth it.

Part 2 of the Condenser is to merge all similar files using least compression
difference. This is decidedly more difficult, since it relies on near-perfect
compression, which is apparently an [AI-hard problem][com]. Unfortunately,
solving AI is out of scope for this post, so I'll leave that for another
time :-)

[key]: https://www.youtube.com/watch?v=lKXe3HUG2l4
[par]: http://dlang.org/phobos/std_parallelism.html
[alg]: http://dlang.org/phobos/std_algorithm.html
[ran]: http://dlang.org/phobos/std_range.html
[com]: http://mattmahoney.net/dc/rationale.html
