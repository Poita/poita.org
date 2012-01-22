---
layout: post
title: Simplicity in Everything
tags:
- meta
- jekyll
- process
---
This website is now created using Jekyll. Originally, I had used Joomla,
a big, industrial-strength content management system.

Joomla worked well for a while, but I quickly reached a point where I
found myself unable to do, what should have been, simple things. For
example, I couldn't see any obvious way of extracting all the text from
my posts -- it was stored in a database somewhere. There was no simple
way to test changes to my site locally because I couldn't see how
everything pieced together. It was just too complex for something that
should have been very simple.

So I got thinking about what would be the simplest way to generate my
website.

Well, what is my website? It's just a collection of pages with the same
layout, but with different blobs of text inserted in the middle for each
post. I'd also like to generate some lists: recent posts, related posts,
that kind of thing.

Ideally, what I want is something that transforms this

{% highlight html %}
    <body>
    <h1>{{ "{{ page.title "}}}}</h1>
    {{ "{{ page.content "}}}}
    </body>
{% endhighlight %}

into this

{% highlight html %}
    <body>
    <h1>Simplicity in Everything</h1>
    <p>This website is now created using Jekyll...</p>
    ...
    </body>
{% endhighlight %}

That's *exactly* what Jekyll does. It just goes through all your pages
and uses [Liquid][1] to transform them into a static site. Don't believe me?
The entire source for this site is [on GitHub][2].

To test my website locally, I just run `jekyll --server` and head on
over to `http://0.0.0.0:4000`. To deploy, I just run `jekyll && rsync
...`, which generates the site and copies it over to my remote server.
That's it.

Why can't everything be this simple?

[1]: http://liquidmarkup.org/
[2]: https://github.com/Poita/poita.org


