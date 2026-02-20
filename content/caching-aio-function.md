Title: Caching an Asyncio Function the Easy Way
Date: 2026-02-16
Category: Blog

If you've used [asyncio](https://docs.python.org/3/library/asyncio.html) for some time you've probably noticed a few things that work differently to the synchronous counter parts.
![How to cache asyncio functions]({static}/images/caching-asyncio.png)
I recently had to add some caching to an asyncio function. Let's say something like:

```py
async def get_user(user_id: UUID) -> User:
    ...
```

Of course naturally we want to use [functools.cache](https://docs.python.org/3/library/functools.html#functools.cache), but past experience have taught me that it doesn't work out of the box. 

Generally the decorators won't work in both sync and async contexts. This is due to the fact that an async function returns immediately when invoked, and is only actually executed when we `await` whatever was returned.


<iframe
  src="https://marimo.app/l/4wydkq?embed=true&show-chrome=false"
  width="100%"
  height="700"
  sandbox="allow-scripts allow-same-origin"
></iframe>

But what actually happens here? What's being cached? How does it work?

Invoking `function()` returns a [coroutine](https://docs.python.org/3/library/asyncio-task.html#coroutines) object which was cached. The object track the state of execution of the async code, `await` will advance the state, it can therefore not be awaited more than once. 

There are other objects that are awaitable, unlike coroutines, [asyncio.Task](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) tracks the result of the coroutine irrespective of the coroutine's state.

<iframe
  src="https://marimo.app/l/fsespx?embed=true&show-chrome=false"
  width="100%"
  height="700"
  sandbox="allow-scripts allow-same-origin"
></iframe>

As shown, this can be awaited as many times as you want, which is exactly what we need here. We can create a simple wrapper around a coroutine function:

```py
def convert_to_task[**P, R](fn: Callable[P, Coroutine[Any, Any, R]]) -> Callable[P, asyncio.Task[R]]:
    """Wrap a coroutine function to make it a task which is cacheable."""

    @wraps(fn)
    def _fn(*args: P.args, **kwargs: P.kwargs) -> asyncio.Task[R]:
        return asyncio.create_task(fn(*args, **kwargs))

    return _fn
```

We can now compose `convert_to_task` with `functools.cache` to make an async function cacheable:
<iframe
  src="https://marimo.app/l/dzij57?embed=true&show-chrome=false"
  width="100%"
  height="700"
  sandbox="allow-scripts allow-same-origin"
></iframe>

### Closing thoughts
Whilst there are off-the-shelf solutions (e.g. [aiocache](https://pypi.org/project/aiocache/)) for this problem, I find that they are missing the simplicity I crave from the functools version. This is probably a signal that we're missing this utility in the standard library.

My solution composes asyncio primitives in a simple way, it's easy enough that you can plausibly reimplement this each time you need it. In lieu of an official version, I believe this is currently the most 'standard' way to achieve caching. 

Finally, though you don't need to understand the difference between a `coroutine` object and a `Task` or even a `Future` object, it helps when you're looking to implement advanced `asyncio` functionality. I was able to drastically simplify the problem by understanding how a `Task` works.
