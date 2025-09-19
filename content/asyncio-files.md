Title: Asyncio - Concurrent file operations
Date: 2025-09-16
Category: Blog
Tags: Asyncio, Concurrency

Asyncio is good for many things, but operations for files are strangely absent from the API. Out of all the different types of 'IO' this is definitely the biggest omission. 

## Why?
Because asyncio's event loop is based on [epoll] on linux which can not provide the necessary functionality to handle all file operations.

## So is there really no way?
Well not exactly, as far as I know there are 2 libraries out there: [aiofile] and [aiofiles]. 

The similar names really don't help here, I imagine this confuses people in the community too. 

### aiofiles

In any case, I've personally used [aiofiles] before, it is more widely known has exactly the same api as the sync versions and uses a thread pool to run the otherwise sync operations in the background.

```py
async with aiofiles.open('filename', mode='r') as f:
    contents = await f.read()
```

### aiofile

[aiofile] on the other hand is something a little more special. It uses linux's aio system calls to achieve truly async file operations. The API is not exactly the same as [aiofiles] but it is still not too bad:

```py
    async with async_open('filename', 'r') as f:
        contents = await f.read()
```

There is also a low level interface that allows for chunked concurrent writes:

```py
async def main():
    async with AIOFile("/tmp/hello.txt", 'w+') as afp:
        await afp.write("Hello ")
        await afp.write("world", offset=7)
        await afp.fsync()
```

### Is io_uring the future?

[io_uring] is a very new interface added in version 5.1. It exposes ring buffers so that the user's application can submit and receive file data without making system calls each time. It should be very efficient and fast. This is the technology that prompted me to look into this topic in the first place.

Unfortunately it has been riddled with controversies since launch. With a number of security issues causing GCP and Docker to disable the feature by default.

The good news is that the recent versions have largely [resolved] the issues. And postgres 18 now supports it [experimentally], with [reported] performance gains.

The bad news is due to the controversies, there's just not been very good Python support for io_uring. I was able to find [uring_file](https://github.com/qweeze/uring_file) that I then [forked](https://github.com/Jamie-Chang/uring_file) to make work with the latest dependencies:

```py
async with open('filename', os.O_RDONLY) as f:
    contents = await f.read()
```

## Benchmark time
I've started doing some benchmarks in a [recent article]({filename}/asyncio-backpressure-followup.md). Using the same memory and speed profiling techniques, I constructed a scenario to write 10,000 lines each concurrently to 100 files.

```py
async def benchmark(
    name: str,
    create: Callable[[File, Iterable[str]], Coroutine[None, None, None]],
    read: Callable[[File], Coroutine[None, None, None]],
    num_files: Limit,
    num_lines: Limit,
) -> None:
    (Path("benchmarks") / name).mkdir(exist_ok=True, parents=True)
    write_name = f"{name}-write"
    with memory_profiler(write_name), timer(write_name):
        async with asyncio.TaskGroup() as tg:
            for i in range(num_files):
                tg.create_task(create(f"benchmarks/{name}/{i}.txt", lines(num_lines)))

    read_name = f"{name}-read"
    with memory_profiler(read_name), timer(read_name):
        async with asyncio.TaskGroup() as tg:
            for i in range(num_files):
                tg.create_task(read(f"benchmarks/{name}/{i}.txt"))
```

The benchmark code takes in a create and a read function and runs them concurrently, then aggregates the time taken. This gives a simple time and memory result, see the following table:




```
aiofile-write: Finished in 31.11s
aiofile-write: Peak memory usage 1,041,145 bytes
aiofile-read: Finished in 0.04s
aiofile-read: Peak memory usage 6,020,660 bytes
aiofiles-write: Finished in 83.52s
aiofiles-write: Peak memory usage 11,113,101 bytes
aiofiles-read: Finished in 80.40s
aiofiles-read: Peak memory usage 2,291,869 bytes
uring-write: Finished in 54.30s
uring-write: Peak memory usage 575,681 bytes
uring-read: Finished in 42.95s
uring-read: Peak memory usage 672,404 bytes
sync-write: Finished in 2.12s
sync-write: Peak memory usage 246,006 bytes
sync-read: Finished in 0.44s
sync-read: Peak memory usage 109,911 bytes
sync-thread-write: Finished in 2.14s
sync-thread-write: Peak memory usage 699,110 bytes
sync-thread-read: Finished in 0.48s
sync-thread-read: Peak memory usage 477,155 bytes
```
