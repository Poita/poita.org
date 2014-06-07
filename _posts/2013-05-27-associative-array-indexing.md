---
layout: post
title: Associative Array Indexing
tags:
- programming
- languages
- associative array
- hashmap
- map
- indexing
- d
- c++
---
I dislike how associative array indexing is handled in every language I know.
In all cases it handled inconsistently with other parts of the language, and in
all cases the inconsistency is unnecessary.

There's two aspects of handling associative arrays that vary among languages:

1. Does `array[key]` automatically insert a value if the key isn't present?
2. Do operations on `array[key]` have "magic" compiler support?

To illustrate these differences, here's how things are handled in C++ and D.

C++
{% highlight c++ %}
std::map<std::string, int> days;
days["Mon"] = 1;
std::cout << days["Tue"] << std::endl;
{% endhighlight %}

D
{% highlight d %}
int[string] days;
days["Mon"] = 1;
writeln(days["Tue"]);
{% endhighlight %}

In C++, `array[key]` always returns a valid lvalue, whether the key was in the
array or not. If the key was not in the array then a default-constructed value is
added, so `days["Tue"]` defaults to `0`.

In D, `array[key]` will return an lvalue only if `key` was in the array. However,
the syntax `array[key] = value` is given magic compiler treatment, and is
converted into a special `opIndexAssign` operator, which is a combination of
indexing and assignment. The attempt to access `days["Tue"]` results in a
range violation.

C++ answers 'yes' and 'no' to questions 1 and 2, and D answers 'no' and 'yes'.
As far as I'm aware, all modern languages follow either the C++ or D approach.
For example, Ruby follows C++ and Python follows D.

Now I'll explain why both of these approaches are bad.

Automatically inserting a default value when the key is present is bad because
it is inconsistent with how normal, non-associative arrays work.

{% highlight c++ %}
std::vector<std::string> days;
days[1] = "Mon"; // ERROR
{% endhighlight %}

Arrays don't automatically expand their domain on indexing, so why should
associative arrays? Aren't arrays just a optimisation of an associative array
where the key is a bounded integer? They should have the same semantics.

And the reason the D approach is bad is because it messes with your intuition
about how expressions work. If I see an indexing operation and an assignment
operation then I expect the AST to have two operation nodes, not one. This has
some interesting consequences:

{% highlight d %}
void set(ref int x, int y) { x = y; }

int[string] days;
days["Mon"] = 1;    // Okay
days["Tue"].set(2); // Error?
{% endhighlight %}

Intuitively, this should work. `set` is no different from the `int` assignment
operator, so I would expect that I can replace all `int` assignments with calls
to `set`. I can't, only because of the compiler magic.

In my opinion, the correct approach is to disallow implicit insertions, and
also avoid any compiler magic. Indexing should be an error when the key isn't
present (just like it is in an array), and if you want to expand the domain of
an associative array then you should need to use a specific operation to do
that (`array.insert` would be the obvious choice).

A likely complaint about this approach would be that you can't write the
three-line word count program, i.e.:

{% highlight d %}
int[string] wc;
foreach (word; words)
    wc[word]++;
{% endhighlight %}

Under my suggested approach, this program would become:

{% highlight d %}
int[string] wc;
foreach (word; words)
    if (word in wc)
    	wc[word]++;
    else
    	wc.insert(word, 1);
{% endhighlight %}

Yes, it's longer, and no, I don't think it's an issue, nor do I think it is an
argument for implicit insertions or magic. If shorter programs are a sound
argument for inconsistent semantics then we're in a lot of trouble. Besides,
it wouldn't be hard to introduce a `lookup` function where you can provide a
default value if the key is missing.

{% highlight d %}
int[string] wc;
foreach (word; words)
    wc.lookup(word, 0)++;
{% endhighlight %}

That's it. Language designers, don't throw away consistency for the sake of
terseness --  it's not worth it!
