---
layout: post
title: D Tip - Beware Narrow Strings
tags:
- d
- dlang
- tip
- strings
- programming
---
In [The D Programming Language][1], strings are arrays. They are literally
aliases to arrays of immutable characters of various width,
[defined in druntime][2].

{% highlight d %}
alias immutable(char)[]  string;
alias immutable(wchar)[] wstring;
alias immutable(dchar)[] dstring;
{% endhighlight %}

All string types in D express [Unicode][3] strings using different encodings.
`string` uses UTF-8, `wstring` uses UTF-16, and `dstring` uses UTF-32.
With the exception of UTF-32, these encodings are _variable length encodings_.
A single "character" may be represented by a variable number of array elements.
These string types are called "narrow strings".

Variable length encoding is very space efficient, but at the cost of indexing.
With `string` and `wstring` there is no way to retrieve the n'th character in
O(1) time. When you use array indexing on narrow strings, you are actually
indexing into the _code units_. For example, the string "こんにちは世界" has 7
code _points_ (characters), but 21 code _units_. `.length` will report 21, and
the element at index 0 is the code unit `227`, which is not こ!

{% highlight d %}
string s = "こんにちは世界";
writeln(s[0] == 'こ'); // false
writeln(s.length); // 21
{% endhighlight %}

As you can see, this behaviour isn't much use when you want to work with the
actual characters. The Phobos developers are aware of this, which is why, when
you treat strings as ranges, they do what you would expect.

{% highlight d %}
string s = "こんにちは世界";
writeln(s.front == 'こ'); // true
writeln(s.walkLength); // 7
{% endhighlight %}

This puts D programmers in a slightly unusual position. Narrow strings are both
arrays of code units, and ranges of code points, depending on how you use them.
When writing generic code, you need to be aware of this because it has some
quite unintuitive consequences:

### T\[\] is not always a range of T
For example, `string` (`immutable(char)[]`) is a range of `dchar`, not `char`.
`typeof("abc".front)` is `dchar`. If you want to store the result of `.front`
then you can use `ElementType!R` (or just use `auto`).

### hasSlicing!T\[\] is not always true
The `hasSlicing!R` trait from `std.range` is `true` when `R` is sliceable. For
strings, it returns `false` because they are only sliceable as arrays of code
units.

### hasLength!T\[\] is not always true
Similarly to above, `hasLength!R` is `true` when you can get the length of `R`
in O(1) time. For strings, it is `false` because you can only the get the
number of code units in O(1) time, not code points. `walkLength` on narrow
strings runs in O(n) time.

### With a T\[\], you can't call .popFront() .length times
`.length` returns the number of code units, but `.popFront()` pops off a code
point, which may be more than one code unit.

In short, try to avoid using arrays directly in generic code. Write your code
to use arbitrary ranges, and add the necessary template constraints when you
want to use array features:

* For indexing, check `isRandomAccessRange!R`.
* For slicing, check `hasSlicing!R`.
* For `.length`, check `hasLength!R`.

If you follow those rules, your generic code should handle narrow strings
just fine.

[1]: http://dlang.org
[2]: https://github.com/D-Programming-Language/druntime/blob/master/src/object.di
[3]: http://en.wikipedia.org/wiki/Unicode