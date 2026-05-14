Title: Free-threading is slowly getting there
Date: 2026-02-20
Category: Blog
Tags: Python, πthon, Python3.14, Free-threading


I posted a lot about free threading in the past year:

- [How free are threads in Python now?]({filename}/free-threading-aoc.md)
- [Python 3.14: State of free threading]({filename}/free-threading-3-14.md)

Over all I had 3 issues with free-threading:

- Reference count contention hurts performance on even immutable objects.
- Often benign code abstractions can hurt multi-threading performance.
- No good profiling support to identify the issues.

Recently I decided to test free-threading performance again, mainly to test the latest 3.15 alpha, but the results surprised me.

<iframe
  src="https://marimo.app/l/q9g49o?embed=true&show-chrome=false"
  width="100%"
  height="700"
  sandbox="allow-scripts allow-same-origin"
></iframe>

Unlike on the pre-released versions, the stable releases of 3.14 actually see massive improvements. We still observe extra reference count contention as we scale up the workers, but it is a lot lower than 3.13. 

The latest alpha of 3.15 sees even more improvements as reference count contention appears to be all but eliminated. Bringing the performance on par with an isolated memory space provided by subinterpreters.

It is clear from the graph that improvements are being made incrementally, but what change caused the massive boost of performance we see from 3.13 to 3.14? 

This may have come from [deferred reference counting](https://github.com/python/cpython/issues/120024). The idea is to delay updating the ref count on an object, therefore avoiding some contention. Unfortunately that's the extent of my current understanding. 

What we can conclude is that, for shared access of builtin data-structures, the performance has improved significantly over the past 2 years. Though there are still some sharp edges which I believe gets in the way of broader adoption, something I will address in the next few posts. 

In any case I hope that with the hard work of the CPython devs we can get to a place where free-threading feels natural and approachable.
