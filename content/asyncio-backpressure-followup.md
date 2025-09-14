Title: Asyncio backpressure - follow up
Date: 2025-09-14
Category: Blog

Previously when discussing [asyncio backpressure]({filename}/asyncio-process-all.md) I've made some claims that were not necessarily complete.

I said:
>It works well for 100s of urls but when we hit a big number like 10000s we have a problem.

>The program seemingly hangs. This is because all the tasks are being created first and only then to do allow the tasks to start executing. The program will also use much more memory than it needs and generally might slow down due to more context switching. Definitely not what we want

There are 2 issues here:

- First about the program hanging, it's actually quite hard to observe that behaviour. 
- Second about the numbers, 10000s of tasks is actually not a big number for asyncio.

This was pointed out in a [Github issue](https://github.com/Jamie-Chang/aiointerpreters/issues/3) by [Misha Behersky](https://github.com/bmwant). Additionally I had not made it very clear why I proposed using semaphore like:

```py
async def process_all(urls: Iterable[str]) -> None:
    semaphore = asyncio.Semaphore(200)
    async with asyncio.TaskGroup() as tg:
        for url in urls:
            await semaphore.acquire()
            tg.create_task(process_one(url)).add_done_callback(lambda _: semaphore.release())

```
As opposed to using it in a more standard way:

```py
def process_one(url, semaphore):
    async with semaphore:
        ...
```

In order to properly explain the behaviour, I needed to construct a much better example so that I can benchmark both memory and speed.


## Simulation
First and foremost, I've been testing back pressure by making concurrent calls to fetch wikipedia articles. 

```py
async def request(client: httpx.AsyncClient, url: str) -> None:
    await client.get(url)
    ...
```

This is a good real world test, but when we are experimenting with thousands of concurrent calls, it adds load to unsuspecting servers. 

I could set up my own servers, but we have a simpler choice.

One of the benefits of asyncio is that it provides primitives for concurrent operations. We can in that case easily simulate the network latency using `asyncio.sleep`:

```py
async def request(url: str) -> None:
    await asyncio.sleep(random.uniform(0.1, 0.2))  # simulate the time delay for getting requests
```

Here we assume the latency is between 100 and 200 ms. But we can also extend this to match a distribution observe in the real world. Similarly we can use this to simulate load with databases and message queues.

## Measuring the speed and memory
Testing the speed or duration is simple, the best way is using [time.perf_counter](https://docs.python.org/3/library/time.html#time.perf_counter). We can compose this into a context manager like this:

```py
@contextmanager
def timer(message: str) -> Iterator[None]:
    start = time.perf_counter()
    try:
        yield
    finally:
        print(message, time.perf_counter() - start, "s")
```

Similarly we can track the peak memory used by using [tracemalloc](https://docs.python.org/3/library/tracemalloc.html):

```py
@contextmanager
def memory_profiler(message: str) -> Iterator[None]:
    tracemalloc.start()

    try:
        yield
        _, peak_memory = tracemalloc.get_traced_memory()
        print(f"Peak memory usage: {peak_memory:,} bytes")
    finally:
        tracemalloc.stop()

```

## Testing different implementations
In the original article, I proposed both batching and semaphore as methods of back pressure. So our test scenarios are as follows 

1. batched execution.
2. acquiring semaphore inside the task as Python intended.
3. acquiring semaphore before task creation and releasing with callback as I proposed.

Finally a bonus implementation suggested in the issue thread to release the semaphore at the end of the function call:

```py
async def main():
    semaphore = Semaphore(...)

    async with TaskGroup() as tg:
        for url in urls:
            await semaphore.acquire()
            tg.create_task(process(url, semaphore))


async def process(url, semaphore):
    try:
        ...
    finally:
        semaphore.release()
```

The code can be found [here](https://github.com/Jamie-Chang/concurreny-bench). We run 100,000 inputs for each implementation limiting concurrency to 500 at a time.

## Results

| File | Description | Memory usage in bytes | Duration in seconds |
|---|---|---|---|
| batching.py | Using batched to process things in batches as opposed to using semaphores | 821,692 | 41.7 |
| semaphore1.py | Traditional pythonic way of using semaphore, does not limit task creation | 152,154,848 |30.9 |
| semaphore2.py | Semaphore limiting task creation with callback | 1,018,796 | 30.2 |
| semaphore3.py | Semaphore limiting task creation with release called in the task function | 839,176 | 30.2 |

## Analysis
Traditional use of semaphore does not limit the number of tasks being created, we must first create all 100,000 tasks and then start processing the tasks. As a result, the amount of memory requires is many times higher. Though the difference in speed is fairly small, it is consistently observable as we are not able to start processing during the creation phase.

Another interesting result is that the semaphore2 with the callback uses more memory than just passing the semaphore to release in semaphore3. This is a quirk of constructing an extra lambda as a callback. Additionally, it may be more obvious than using the callback, so this method is definitely worth considering.

Finally I want to talk about batching. Batching is very memory efficient as it doesn't need any extra objects however it does take a lot longer.
This is explained in my original post:

> The problem is if we have a small amount of tasks with long wait times then it'll slow down the whole batch.

Notice that our distributions has a lot of variance:
```py
random.uniform(0.1, 0.2)
```

if we reduce this to `random.uniform(0.1, 0.11)` we have closer results


| File | Duration in seconds |
|---|---|
| batching.py | 24.65 |
| semaphore1.py | 22.0 |
| semaphore2.py | 21.1 |
| semaphore3.py | 21.1 |

My view is batching and the 2 semaphore solutions are all very good solutions. Batching is simple, but you can get more performance out of using semaphores just not in the obvious way.

On the other hand using semaphore normally is good but you should be aware of the memory implication.

## Finally
I've spent a lot of time investigating asyncio back pressure here. The point is to provide the full context of the different options to use. 

I also hope that I've provided some ideas around how you might investigate different design patterns yourself.


