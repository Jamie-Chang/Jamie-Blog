Title: Free Threaded Python With Asyncio
Date: 2024.09.18
Category: Blog

With the immanent release of Python 3.13, I wanted to look at the biggest changes coming to Python. I think by far the most exciting feature is free-threaded Python from [PEP-703](https://peps.python.org/pep-0703/).

As I'm quite late to the party, there's already a of articles talking about it. I came accross an excellent [article](https://til.simonwillison.net/python/trying-free-threaded-python) from Simon Wilson, which successfully demonstrated parallelism for pure Python functions. Building on top of this I wanted to look at ways to synchronise threads beyond using `ThreadPool.map`. 

Prior to Python 3.13 threads were used for IO bound tasks due to the GIL, Asyncio is also used for IO (duh...) and we can wrap threads using [`asyncio.to_thread`](https://docs.python.org/3/library/asyncio-task.html#asyncio.to_thread). For example,

```python
await asyncio.to_thread(io_bound_task, "first_arg", optional="optional")
```
Here's a quote lifted directly from Asyncio docs. In our case the GIL is not a problem:

> Note Due to the GIL, asyncio.to_thread() can typically only be used to make IO-bound functions non-blocking. However, for extension modules that release the GIL or alternative Python implementations that don’t have one, asyncio.to_thread() can also be used for CPU-bound functions.

So I modified Simon's example:

```python
import argparse
import os
import sys
import time
from asyncio import get_running_loop, run, to_thread, TaskGroup
from concurrent.futures import ThreadPoolExecutor
from contextlib import contextmanager


@contextmanager
def timer():
    start = time.time()
    yield
    print(f"Elapsed time: {time.time() - start}")


def cpu_bound_task(n):
    """A CPU-bound task that computes the sum of squares up to n."""
    return sum(i * i for i in range(n))


async def main():
    parser = argparse.ArgumentParser(description="Run a CPU-bound task with threads")
    parser.add_argument("--threads", type=int, default=4, help="Number of threads")
    parser.add_argument("--tasks", type=int, default=10, help="Number of tasks")
    parser.add_argument(
        "--size", type=int, default=5000000, help="Task size (n for sum of squares)"
    )
    args = parser.parse_args()

    get_running_loop().set_default_executor(ThreadPoolExecutor(max_workers=args.threads))

    with timer():
        async with TaskGroup() as tg:
            for _ in range(args.tasks):
                tg.create_task(to_thread(cpu_bound_task, args.size))


if __name__ == "__main__":
    print("Parallel with Asyncio")
    print(f"GIL {sys._is_gil_enabled()}")  # type: ignore
    run(main())
```


