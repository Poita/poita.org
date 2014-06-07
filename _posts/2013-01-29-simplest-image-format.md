---
layout: post
title: Simplest Image Format
tags:
- ppm
- image format
- programming
---
Just a quick post to share something I discovered a few days ago: the simplest
image format ever. I'm actually quite surprised it took me so long to find it,
which is why I'm sharing now.

The format I'm talking about is the [Portable Pixmap Format (.ppm)][1]. It's
so simple to write that it's easier to describe using code:

{% highlight d %}
// --------------------------------------------------------------------
//     file - the file to write to
//    width - width of the image in pixels
//   height - height of the image in pixels
// rgb_data - the image data, 3 bytes per pixel (R, G, B)
// --------------------------------------------------------------------
void write_ppm(FILE* file, int width, int height, void* rgb_data)
{
    fprintf(file, "P6 %d %d 255\n", width, height);
    fwrite(rgb_data, 1, 3 * width * height, file);
}
{% endhighlight %}

That's it: a small header followed by the raw image data.

It's incredibly useful to have a simple file format like this when you want
to write out some images at runtime and don't have any other image writing
libraries available (or can't be bothered hooking them up).

[1]: http://en.wikipedia.org/wiki/Netpbm_format