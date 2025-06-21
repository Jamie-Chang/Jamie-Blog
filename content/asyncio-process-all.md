Title: Asyncio backpressure - Processing lots of tasks in parallel
Date: 2025-06-20
Category: Blog

There's mixed feedback to Asyncio in the community. Some people passionately hate it whilst others believe "writing async make program go fast". This debate is way too much for me to cover here right now. Though maybe I'll look at it in the future. 

For me I use asyncio a lot, and it's genuinely a useful tool but not without issues. I have had a problem for a while, it involves fetching a large amount of urls:

```py
async def process_one(url: str) -> None:
    data = await fetch(url)
    await upload(data)


async def process_all(urls: Iterable[str]) -> None:
    for url in urls:
        await process_one(url)
```

But since it's asyncio we're using then we want to parallelise the tasks:

```py
async def process_all(urls: Iterable[str]) -> None:
    async with asyncio.TaskGroup() as tg:
        for url in urls:
            tg.create_task(process_one(url))
```

Now here's where the problem starts. It works well for 100s of urls but when we hit a big number like 10000s we have a problem.

The program seemingly hangs. This is because all the tasks are being created first and only then to do allow the tasks to start executing. The program will also use much more memory than it needs and generally might slow down due to more context switching. Definitely not what we want!

### sleep
At a first glance, we could instead add a bit of a `sleep` in the for loop allowing some tasks to start. And easing the pressure on the system.

```py
async def process_all(urls: Iterable[str]) -> None:
    async with asyncio.TaskGroup() as tg:
        for url in urls:
            await asyncio.sleep(0)
            tg.create_task(process_one(url))
```
Though we can adjust the sleep according to the speed of processing, in practice the number is quite hard to pick. Too much sleep and you slow down too much. 


### batch
So rethinking the problem, we can act more directly. What we really want is to simply reduce the amount of tasks that can be created in parallel. 

A simple way to do so is by batching. [`itertools.batched`](https://docs.python.org/3/library/itertools.html#itertools.batched)
makes it easy to split the urls into manageable batches:

```py
async def process_all(urls: Iterable[str]) -> None:
    for batch in batched(urls, 200):
        async with asyncio.TaskGroup() as tg:
            for url in batch:
                tg.create_task(process_one(url))
```
We allow 200 tasks to run in parallel and wait until all tasks are complete before starting the next 200.

This is simple and easy to understand, as a results it's been my go to method for problems like this. The problem is if we have a small amount of tasks with long wait times then it'll slow down the whole batch. 

### Semaphore
Finally that leads me to `asyncio.Semaphore`. `Semaphore` is a common tool to limit the number of concurrent accesses to a resource. Generally the usecase is to add it inside of the task:

```py
semaphore = asyncio.Semaphore(200)


async def process_one(url: str) -> None:
    async with semaphore:
        data = await fetch(url)
        await upload(data)
```
This will limit the number of concurrent fetches and uploads. But this doesn't protect us from creating the tasks in the first place. So we need to use it during task creation:

```py
async def process_all(urls: Iterable[str]) -> None:
    semaphore = asyncio.Semaphore(200)
    async with asyncio.TaskGroup() as tg:
        for url in urls:
            await semaphore.acquire()
            tg.create_task(process_one(url)).add_done_callback(lambda _: semaphore.release())
```

The `semaphore.acquire` call will block us from creating the tasks before the one of the tasks finishes and releases the semaphore. So we'll have a pretty constant 200 concurrent tasks until we start exhausting the urls.

## Backpressure
The techniques mentioned above are all a form of [backpressure](https://en.wikipedia.org/wiki/Back_pressure). We're trying to ease the memory pressure of the system by either sleeping to slow down the rate of task creation or to place some physical limits on the number of tasks that can be created.

The exact mechanism will differ by use case and runtime. For example, we don't need to explicitly create backpressure for threadpools as the concurrency is already limited by the pool size: 

```py
def process_one(url):
    ...


def process_all(urls: Iterable[str]) -> None:
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as pool:
        pool.map(process_one, urls)
```

[`Queues`](https://docs.python.org/3/library/asyncio-queue.html#asyncio.Queue) with `max_size` are another method that comes to mind. And generally your use case might differ and require other mechanisms. Backpressure is a topic that's not covered particularly well and it's definitely worth playing around with different methods for your own code. 