Title: Python 3.15: features that didn't make the headlines
Date: 2026.05.13
Category: Blog
Tags: Python, Python3.15

It's that time of the year again, a new version of Python is just around the corner. With the [Python 3.15.0b1](https://docs.python.org/3.15/whatsnew/3.15.html) feature freeze, we know what's coming to Python later this year. There are so many big features coming including [lazy imports](https://docs.python.org/3.15/whatsnew/3.15.html#whatsnew315-lazy-imports) and the [tachyon profiler](https://docs.python.org/3.15/whatsnew/3.15.html#whatsnew315-sampling-profiler) which I [previously covered]({filename}/free-threading-benchmark.md).

Last year, I really enjoyed [investigating]({filename}/python-3-14.md) the smaller features of Python 3.14. I found that many of those features were just as interesting as the big PEPs and deserve a lot more attention. This year the situation is no different.


## Asyncio Taskgroup Cancellation
There are not many Asyncio changes in this releases. The main feature to come out here is the ability to cancel a `TaskGroup` gracefully.

[TaskGroup](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup) is a form of [structured concurrency](https://en.wikipedia.org/wiki/Structured_concurrency), it enables developers to create multiple concurrent tasks in a clean way. 

```py
async with asyncio.TaskGroup() as tg:
    tg.create_task(run())
    tg.create_task(run())
    # Waits for all the tasks to complete
```


Suppose we want to wait in the background for a signal of sorts to interrupt the taskgroup's execution, it's seems like something simple to do in asyncio, but in reality it's somewhat awkward to do this.

```py
class Interrupt(Exception):
    ...


with suppress(Interrupt):
    async with asyncio.TaskGroup() as tg:
        tg.create_task(run())
        tg.create_task(run())
        if await wait_for_signal():
            raise Interrupt()

```

This works because exceptions raised within a task group cause other tasks to cancel. The custom `Interrupt` exception is raised as part of a `ExceptionGroup` which then gets filtered by [contextlib.suppress](https://docs.python.org/3/library/contextlib.html#contextlib.suppress), resulting in a graceful exit. 

> The way suppress works with ExceptionGroup is yet another overlooked feature from 3.12. This is a change I learnt by accident when researching this article.


The new [TaskGroup.cancel](https://docs.python.org/3.15/library/asyncio-task.html#asyncio.TaskGroup.cancel) makes this process a lot easier: 

```py
async with asyncio.TaskGroup() as tg:
    tg.create_task(run())
    tg.create_task(run())
    if await wait_for_signal():
        tg.cancel()
```

Unlike before it's so simple there's hardly any point in explaining. It simply cancels the group without raising any exceptions.

## Context Manager Improvements
Decorators are surprisingly hard to write, so much so that it's become a go-to interview question. But did you know that context managers can also double up as a decorator?

```py
@contextmanager
def duration(message: str) -> Iterator[None]:
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{message} elapsed {time.perf_counter() - start:.2f} seconds")
```
Here I have a very commonly used context manager to print out the duration spent in the block. Ever since Python 3.3 we could directly use it as a decorator too:

```py
@duration('workload')
def workload():
    ...


# Or simple as a wrapper

duration('stuff')(other_workload)(...)
```

But whilst it's convenient, there are cases where it doesn't work at all: 

```py
@duration('async workload')
async def async_workload():
    ...


@duration('generator workload')
def workload():
    while True:
        yield ...
```

Iterators, async functions and async iterators don't work well here because they have different semantics to standard functions. When you call them they return immediately with a generator object, coroutine function and async generator object respectively. So the decorator completes immediately as opposed to the entire lifecycle what it's wrapping.

This is an unfortunate problem I've encountered many times, and it's often a problem for normal decorators too. But this has changed in 3.15, now the `ContextDecorator` will check the type of the function it's wrapping and ensure that the decorator covers the entire lifespan. 

In my opinion, this now makes context managers the best way to create decorators! It avoids some of the common footguns and provides cleaner syntax. I recommend more people start using it this way.

## Thread Safe Iterators
Iterators are one of the foundations of modern Python. The iterator type allows us to separate data sources from data consumers as below, resulting in cleaner abstractions: 

```py
lazy from typing import Iterator

def stream_events(...) -> Iterator[str]:
    while True:
        yield blocking_get_event(...)

events = stream_events(...)

for event in events:
    consume(event)
```

But this abstraction breaks when using threading or free-threading. An iterator by default is not threadsafe, therefore we may see skipped values or just broken internal iterator state. 

This is solved in 3.15 with [threading.serialize_iterator](https://docs.python.org/3.15/library/threading.html#threading.serialize_iterator), we simply wrap our original iterator with this and voila: 

```py
import threading

events = threading.serialize_iterator(stream_events(...))

with ThreadPoolExecutor() as executor:
    fut1 = executor.submit(consume, events)
    fut2 = executor.submit(consume, events)
```

There is also the [threading.synchronized_iterator](https://docs.python.org/3.15/library/threading.html#threading.synchronized_iterator) decorator which just applies `threading.serialize_iterator` to the result of an generator function. 


Finally we also have [threading.concurrent_tee](https://docs.python.org/3.15/library/threading.html#threading.concurrent_tee) that instead of splitting the values will duplicate the values across multiple iterators: 

```py
source1, source2 = threading.concurrent_tee(squares(10), n=2)

with ThreadPoolExecutor() as executor:
    fut1 = executor.submit(consume, source1)
    fut2 = executor.submit(consume, source2)
```

Before these utilities existed we primarily relied on [Queue](https://docs.python.org/3/library/queue.html)s to synchronise consumption between threads, with these added in we can avoid changing our abstractions for multi-threaded code.

## Bonus Features
Last year I only highlighted 3 features, but this year there are a lot more updates that intrigue me. Here are 2 more changes that are perhaps less impactful but still very interesting nonetheless. 

### Counter xor Operation
[collections.Counter](https://docs.python.org/3/library/collections.html#collections.Counter) is a very useful class. It let's us easily count up the frequency of discrete occurrences. It behaves very similar to a `dict[KeyType, int]` but with a ton of useful operations


```py
c = Counter(a=3, b=1)
d = Counter(a=1, b=2)
print(f"{c + d = }")  # add two counters together:  c[x] + d[x]
print(f"{c - d = }")  # subtract (keeping only positive counts)
```

prints:

```
Counter(a=4, b=3)
Counter(a=1, b=0)
```

But it has some weirder operations too:

```py
print(f"{c & d = }")  # intersection:  min(c[x], d[x])
print(f"{c | d = }")  # union:  max(c[x], d[x])
```

prints: 

```
Counter(a=1, b=1)
Counter(a=3, b=2)
```

The way to think of it is that a `Counter` can also represents a discrete set of objects. so in our example, we're essentially doing: 

```py
{a_0, a_1, a_2, b_0} & {a_0, b_0, b_1} == {a_0, b_0}
{a_0, a_1, a_2, b_0} | {a_0, b_0, b_1} == {a_0, a_1, a_2, b_0, b_1}
```

In 3.15 we can also add xor to the list: 

```py
c = Counter(a=3, b=1)
d = Counter(a=1, b=2)

c ^ d == c | d - c & d == Counter(a=3, b=2) - Counter(a=1, b=1) == Counter(a=2, b=1)
```

Once again this is best explained by our notation from earlier: 

```py
{a_0, a_1, a_2, b_0} ^ {a_0, b_0, b_1} == {a_1, a_2, b_1}
```

I've left this one to the bonus section because I've never used set operations on `Counter`s and I'm finding it extremely hard to think of a use case for xor specifically. But I do appreciate the devs adding it for completeness.


### Immutable JSON Objects
With the addition of [frozendict](https://peps.python.org/pep-0814/) in 3.15, we now have the ability to represent all the json types (array, boolean, float, null, string, object) in immutable (hashable) forms. 

A change has been made to [json.load](https://docs.python.org/3.15/library/json.html#json.load) and [json.loads](https://docs.python.org/3.15/library/json.html#json.loads) to add `array_hook` parameter that compliments the `object_hook` parameter. This now allows us to parse json objects directly into this form:

```py
json.loads('{"a": [1, 2, 3, 4]}', array_hook=tuple, object_hook=frozendict) == frozendict({'a': (1, 2, 3, 4)})
```


