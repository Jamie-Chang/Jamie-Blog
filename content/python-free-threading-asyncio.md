Title: Free Threaded Python With Asyncio
Date: 2024.09.18
Category: Blog

The release of Python 3.13 is immanent, the most exciting feature is free-threaded Python from [PEP-703](https://peps.python.org/pep-0703/). 

I see a lot of activity online testing out the new Python build. I recently came across an excellent post on this, so I won't go into much detail on setting this up.

I've been interested in different ways we can dispatch many threads. Prior to Python 3.13 threads were used for IO bound tasks due to the GIL. Asyncio also allows us to manage these types of tasks, and Python has a way to launch threads via Asyncio since Python 3.9:

await asyncio.to_thread(io_bound, 10000)

I wanted to see if we can do thr same for cpu bound workloads in 3.13. So i modified the example:


