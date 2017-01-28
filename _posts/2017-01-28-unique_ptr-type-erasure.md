---
layout: post
title: unique_ptr Type Erasure
tags:
- cpp
- type-erasure
- unique_ptr
---
Often when building APIs we'd like the caller to provide some information in
a format, or using a protocol defined by the library. For example, we might
want access to a contiguous buffer specified by a (`void*`, `size_t`) pair.

{% highlight cpp %}
void cool_feature(void* buffer, size_t len);
{% endhighlight %}

An API like this is fine if the buffer will be read synchonously, but
sometimes we need asynchronous access. This introduces a lifetime problem:
how does the caller know when it can free the buffer?

A common way to solve this is to provide a callback that will be invoked once
the processing has completed:

{% highlight cpp %}
void cool_feature(void* buffer, size_t len, void (*done)(void*));
{% endhighlight %}

Here, `done` will be called with `buffer` once `cool_feature` has asynchonously
finished its work.

This works most of the time, but has a couple of problems:

1. You need to remember to call `done` in all cases.
2. It only works if done only needs to be called with the buffer pointer. What
if my buffer is part of a larger object?

{% highlight cpp %}
struct Object {
  // ...
  std::string buffer;
  // ...
};

void do_stuff(std::unique_ptr<Object> obj) {
  cool_feature(obj.buffer.data(), obj.buffer.size(), ???);
}
{% endhighlight %}

There's no obvious choice of function to pass in. We want to free `obj` once
buffer processing is done, but we'll be given a pointer to the string data, not
`obj`.

A possible solution is to pass in an `std::function<void()>` as the callback,
so that `obj` can be captured and destroyed as necessary. This works, but
`std::function` has difficulty with move-only types, and there's no guarantees
that a capturing `std::function` won't allocate memory.

We could, of course, define another interface (maybe call it
`ContiguousBufferProvider`), but this also requires an extra memory allocation.

A little trick we can do to handle most of the common cases is to pass in
a type-erased `context` to the function.

{% highlight cpp %}
using Context = std::unique_ptr<void*, void(*)(void*)>;

void cool_feature(void* buffer, size_t len, Context context);
{% endhighlight %}

The context is essentially an extra piece of baggage that must be carried
around until the buffer has been processed. Once processed, we just drop the
context and it cleans up after itself.

We can also benefit from an extra function that converts any object into
a `Context` through type erasure:

{% highlight cpp %}
template <typename T>
void deleter(void* p) {
  delete static_cast<T*>(p);
}

template <typename T>
Context erase_type(std::unique_ptr<T> p) {
  return Context(static_cast<void*>(p.release()), &deleter<T>);
}
{% endhighlight %}

With this, the `Object` example is easy to solve:

{% highlight cpp %}
void do_stuff(std::unique_ptr<Object> obj) {
  cool_feature(
      obj.buffer.data(),
      obj.buffer.size(),
      erase_type(std::move(obj));
}
{% endhighlight %}

This guarantees no extra memory allocations, and `cool_feature` just needs to
drop the context once finished.

There are probably more general ways to solve this, but I like this approach as
it is clean, simple, and solves all the problems I've encountered so far.
