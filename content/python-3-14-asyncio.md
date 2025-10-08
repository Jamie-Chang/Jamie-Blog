Title: Python 3.14: 3 Asyncio Changes
Date: 2025.10.09
Category: Blog
Tags: Python, πthon, Python3.14, asyncio

Python 3.14 was officially released on October 7th. There are a lot of new features and I've covered some of them before in:

- [Python 3.14: 3 smaller features]({filename}/python-3-14.md)
- [t-strings: the good and the ugly]({filename}/t-strings.md)
- [Subinterpreters and Asyncio]({filename}/aiointerpreters.md)

What I haven't covered here are any of the asyncio changes. There just happened to also be 3 changes to Asyncio this release ... 


### Asyncio Debugger
Python's builtin debugger [`pdb`](https://docs.python.org/3/library/pdb.html) may seem basic but one of its most powerful features is to run python code at a breakpoint:

```py
def my_func(data: dict) -> None:
    breakpoint()  # or import pdb; pdb.set_trace()
    ...
```

Which enters the debugger at the stack frame where breakpoint is called and allows us to interact with objects in scope. 
```
>>> my_func({})
> /Users/jamie.chang/projects/Jamie-Blog/<stdin>-0(2)my_func()
-> import sys
(Pdb) data
{}
(Pdb) 
```

However when it comes to asyncio commands it gets tricky:

```py
class MyClient:
    async def now(self):
        return time.time()


async def main():
    client = MyClient()
    breakpoint()
    ...
```

```
(Pdb) client.now()
<coroutine object MyClient.now at 0x102ad16c0>
```

We cannot await at all in pdb:
```
(Pdb) await client.now()
*** SyntaxError: 'await' outside function
```

Nor can we run it with `asyncio.run`:
```
(Pdb) asyncio.run(client.now())
*** RuntimeError: asyncio.run() cannot be called from a running event loop
```

Now this is possible with [pdb.set_trace_async](https://docs.python.org/3.14/library/pdb.html#pdb.set_trace_async)

```py
async def main():
    client = MyClient()
    await pdb.set_trace_async()
    ...
```

> Note that `set_trace_async()` is awaited here. As I understand it, this is a necessary compromise to get this feature working.

And ... Success!
```
(Pdb) print(await client.now())
1759933736.955492
```

This might be a niche feature, but it's crucial for debugging things like async ORMs.

If you just need to interact with async resources in a REPL, you can already start an async REPL since Python 3.8:

```bash
$ python -m asyncio
asyncio REPL 3.9.6 (default, Nov 11 2024, 03:15:38) 
[Clang 16.0.0 (clang-1600.0.26.6)] on darwin
Use "await" directly instead of "asyncio.run()".
Type "help", "copyright", "credits" or "license" for more information.
>>> import asyncio
>>> await asyncio.sleep(1)
```

### Asyncio Introspection
Whether I'm teaching asyncio to newcomers or debugging complex concurrent programs myself. I've always found it difficult to visualise what's happening. 

> When teaching asyncio we often just time the difference in performance, explaining it properly is hard.

It is now possible with [introspection](https://docs.python.org/3.14/whatsnew/3.14.html#asyncio-introspection-capabilities). We can see the current call graph at runtime by:

```bash
python -m asyncio pstree 12345
```

where 12345 is the process id.

> Note on macOS it needs to be run using sudo.

The graph looks like:

```
└── (T) Task-1
    └──  main /Users/jamie.chang/projects/concurreny-bench/backpressure/semaphore3.py:25
        └──  Semaphore.acquire /Users/jamie.chang/.local/share/uv/python/cpython-3.14.0rc3-macos-aarch64-none/lib/python3.14/asyncio/locks.py:407
            ├── (T) Task-30376
            │   └──  process_url /Users/jamie.chang/projects/concurreny-bench/backpressure/semaphore3.py:15
            │       └──  sleep /Users/jamie.chang/.local/share/uv/python/cpython-3.14.0rc3-macos-aarch64-none/lib/python3.14/asyncio/tasks.py:702
            ├── (T) Task-30389
            │   └──  process_url /Users/jamie.chang/projects/concurreny-bench/backpressure/semaphore3.py:15
            │       └──  sleep /Users/jamie.chang/.local/share/uv/python/cpython-3.14.0rc3-macos-aarch64-none/lib/python3.14/asyncio/tasks.py:702
            ├── (T) Task-30393
            │   └──  process_url /Users/jamie.chang/projects/concurreny-bench/backpressure/semaphore3.py:15
            │       └──  sleep /Users/jamie.chang/.local/share/uv/python/cpython-3.14.0rc3-macos-aarch64-none/lib/python3.14/asyncio/tasks.py:702
            ├── (T) Task-30408
            │   └──  process_url /Users/jamie.chang/projects/concurreny-bench/backpressure/semaphore3.py:15
            │       └──  sleep /Users/jamie.chang/.local/share/uv/python/cpython-3.14.0rc3-macos-aarch64-none/lib/python3.14/asyncio/tasks.py:702
```

And make it a lot clearer what's going on inside the event loop. 

Since it can be triggered on a currently running process, we can even use this to debug a live running application that may be stuck on some async resources. 

I've been using this with [watch](https://en.wikipedia.org/wiki/Watch_(command)) to see a live view of the graph.

You can also get the information in table form by:

```bash
python -m asyncio ps 12345
```

Finally, there are new [python apis](https://docs.python.org/3.14/library/asyncio-graph.html) for this to get the information at a specific point in the code.

### Asyncio in multiple free threads
Not to be confused with [Free Threaded Python With Asyncio]({filename}/python-free-threading-asyncio.md). Where we run specific blocking code in threads using [asyncio.to_thread](https://docs.python.org/3/library/asyncio-task.html#asyncio.to_thread).

Prior to 3.14 event loops were not thread safe in the free-threaded builds of Python. This means you can't run asyncio in a separate thread. In Python 3.14 Kumar Aditya updated the event loop implementation maing it thread safe whilst also improving performance. You can see his blog post [here](https://labs.quansight.org/blog/scaling-asyncio-on-free-threaded-python).

But if we are already doing things concurrently why would threads matter? Well this actually depends on how much blocking work your coroutines are doing.

Let's say we have N coroutines all fetching data from a web page somewhere. This is perfect for asyncio, as http requests can be made without blocking the event loop.

```py
async with TaskGroup() as tg:
    for url in urls:
        tg.create_task(scrape(url))
```

But what do we do with the scraped data? We need to parse this, so this will block the event loop as it's CPU bound.

```py
async def process(url):
    data = await scrape(url)
    parse(data)  # CPU bound
    ...

...

async with TaskGroup() as tg:
    for url in urls:
        tg.create_task(process(url))
```

You will ind that the CPU bound tasks will impact the performance of the event loop. This is actually a situation that I've run into a couple times already. This can be solved with `asyncio.to_thread` here but sharing a lot of data right now may have a [performance impact]({filename}/free-threading-aoc.md). 

Instead we can now spawn multiple free-threads with event loops. Each thread is still impacted by the CPU bound blocking tasks, however as we are processing the CPU-bound work concurrently, the impact is lowered. 

We will need a way to pass data between the threads, the best way I found is using [queue.Queue](https://docs.python.org/3/library/queue.html#queue.Queue):

```py
async with asyncio.TaskGroup() as tg:
    while True:
        try:
            url = queue.get_nowait()
        except Empty:
            break
        task = tg.create_task(process(url))
        task.add_done_callback(lambda _: queue.task_done())
```


For benchmark performances it's better to check Kumar's blog post. He posts a very impressive performance improvement, which I also see in my own examples. He also shows a much higher raw TCP performance so there's potential to increase IO bound workloads as well.

On the other hand, I found free-threading memory sharing awkward right now in Python and that impacts performance. My preferred method is using my very own [aiointerpreters](https://github.com/Jamie-Chang/aiointerpreters/blob/main/examples/crawl.py) to launch the tasks to interpreter threads with subinterpreter's powerful shared memory functionality. 

As with a lot of concurrency problems, there is not a single way to solve these. This is fine, and you should try different approaches, run benchmarks see how they compare to pick the best method for your problem. 
