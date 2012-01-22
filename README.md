This repo contains the source for my website: [poita.org][1]

The site is statically generated using [Jekyll][2].

To generate the site:

    git clone git@github.com:Poita/poita.org.git poita.org
    cd poita.org
    jekyll --server

Then head to `http://0.0.0.0:4000/` to view locally.

To deploy, just copy the contents of `poita.org/_site` (generated by
jekyll) to your server's document root. I use [rsync][3] for this.

[1]: http://poita.org
[2]: https://github.com/mojombo/jekyll
[3]: http://en.wikipedia.org/wiki/Rsync
