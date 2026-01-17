Title: Asyncio is neither fast nor slow 
Date: 2026-01-12
Category: Blog

***Don't listen to random benchmarks..***


I recently came across an [article](https://hackeryarn.com/post/async-python-benchmarks/) benchmarking Python performances in web frameworks, comparing asyncio and sync performance. 

The author sets out to measure performance of [FastAPI](https://fastapi.tiangolo.com/)/[Django](https://www.djangoproject.com/) web servers running with postgresql comparing async and non-async workloads. The methodology is pretty reasonable, the following is his result for an endpoint with a single postgres database read:


Server type|workers|RPS|Latency avg|Latency max|Median
---|---|---|---|---|---
Sync Django|1|456|140ms|262ms|153ms
Sync Django|2|669|96ms|262ms|132ms
Sync Django Pooled|1|569|112ms|171ms|117ms
Sync Django Pooled|2|1822|35ms|98ms|50ms
Async Django|1|205|312ms|467ms|331ms
Async Django|2|541|118ms|304ms|196ms
FastAPI|1|236|271ms|372ms|287ms
FastAPI|2|409|156ms|433ms|224ms


As a result the author concludes:
> These benchmarks show just how much optimization for sync web services Django and the Python has. Sync Django Pooled outperforms or matches all other configurations. Even FastAPI only performs better when it’s the sole bottleneck.

When I read these result I had some doubts, I noticed that:  

1.⁠ ⁠it shows that the best sync django scenario with 2 workers, has more than 4x throughput that the equivalent fastapi async solution.
2.⁠ ⁠2 workers of sync django pooled is 3x the performance of a single worker.

The former doesn't line with the my personal experience. Asyncio does add overhead in some cases, but the scenario laid out here should have postgres as the bottleneck. Overhead caused by asyncio's event loop shouldn't be significant. 

The latter also seemed a little odd, as the throughput increase should be proportional to the number of workers.

### Reviewing the code
Since the results don't match my own expectation then either my mental model is wrong or it's the benchmark. So let's review the benchmark code.

The code for the fastapi endpoint is as follows:

```py
@app.get("/quote")
async def quote(db: AsyncSession = Depends(get_db)):
    statement = (
        select(Quote).order_by(func.random()).options(selectinload(Quote.author))
    )
    result = await db.execute(statement)
    quote = result.scalar()

    return {
        "quote": quote.quote_text,
        "author": quote.author.name,
    }
```


Here a single random quote is fetched from the table. [SQLAlchemy](https://www.sqlalchemy.org/) is used as an orm to facilitate query building. Everything seems in order.

But wait, there's a subtle problem here! By default [SQLAlchemy](https://www.sqlalchemy.org/) does not batch results, instead it will fetch all results into memory first when ⁠ execute ⁠ is called. So whilst ⁠ `.scalar()` ⁠only returns the first result, in the background all results are fetched.

### Recreating the benchmark
So I recreated the benchmark with my code fix for fastapi.

```python
statement = (
    select(Quote).order_by(func.random()).limit(1).options(selectinload(Quote.author))
)
```
I tried where possible to keep the same code and parameters to the original. 

•⁠  ⁠Hardware: Macbook Pro M3 11 cores 18 GB of RAM
•⁠  ⁠Connection Pool Configuration: min 5 - max 15 connections
•⁠  ⁠Data: 100 authors with 1000 quotes
•⁠  ⁠Benchmark: ⁠ rewrk -d 30s -c 64 --host http://localhost:8000/quote/ ⁠

### Results
And the results are as follows:

Server type|workers|RPS|Latency avg|Latency max|Median
---|---|---|---|---|---
Sync Django|1|341.81|186.61ms|418.90ms|223.96ms
Sync Django|2|338.74|188.07ms|467.42ms|227.23ms
Sync Django Pooled|1|440.46|144.92ms|281.12ms|150.94ms
Sync Django Pooled|2|419.01|152.30ms|374.59ms|167.25ms
Async Django|1|323.32|197.28ms|497.76ms|249.85ms
Async Django|2|322.78|197.45ms|453.63ms|249.18ms
FastAPI|1|687.26|92.95ms|326.49ms|110.29ms
FastAPI|2|673.85|94.78ms|332.11ms|121.00ms

As expected the FastAPI implementation does a lot better here compared to before. Performing better than the best django version. 

What's weird is that the worker count makes very little difference in the performance, often slowing down rather than speeding up. My guess is that my setup has more rows in the database, therefore we reach a saturation point a lot quicker.


### The dilemma of benchmarks
Still you might be a bit disappointed with this benchmark as I was, the original had a much clearer trend between async/sync worker count and pooled database connections. However this is precisely what happens with these "real world" benchmarks. Here we're trying to represent a real use case, not necessarily to prove a point. 

On the other hand, there exist many micro-benchmarks out there, where variables are controlled and a smaller more predictable scenario is measured.

There are places for both types of benchmarks, real world benchmarks can be more interesting but the data is messy and requires more critical analysis.

### Critical analysis
My results are convenient for my beliefs and one might conclude from it that asyncio is indeed much faster. 

But looking at the numbers I wondered why Django is so much slower, where does the slow down actually come from. I expected asyncio to do quite well but I assumed we can make up the performance with more threads or workers, but that does not seem to be the case.

This lead me into a deep dive, trying almost every combination of configuration, implementation and framework. 

First I wanted more control over the variables, [Django](https://www.djangoproject.com/) is a very different framework compared to [FastAPI](https://fastapi.tiangolo.com/) and a there are too many possibilities when it comes to the performance discrepancy. [Flask](https://flask.palletsprojects.com/) is a lighter weight framework compared to [Django](https://www.djangoproject.com/) as such it is a much better sync vs async comparison.

Secondly, the original benchmark makes a point of the benefits of pooled connections. However, async django can also benefit from pooled connections. [Sqlalchemy](https://www.sqlalchemy.org/) also uses a connection pool by default. So I see no reason to not use connection pools across the board.

### The new results
This time around I decided to run different thread and worker configurations through the gauntlet and only keep the best performing configurations.

This was generally 3 workers and 5 threads on my machine, giving me a total of 15 threads. However other configurations with similar total thread counts also performed very similarly. 

Server type|workers|threads|RPS|Latency avg|Latency max|Median
---|---|---|---|---|---|---
FastAPI (Pooled)|1|-|687.26|92.95ms|326.49ms|110.29ms
Flask (Pooled)|3|5|682.73|93.66ms|239.67ms|104.34ms
Django (Pooled)|3|5|411.91|155.17ms|299.19ms|169.44ms
Django Async (Pooled)|1|-|406.90|152.79ms|279.93ms|160.32ms

> NOTE: asgi for the async frameworks don't use threads

And this time, the results are more even with the fastapi implementation pulling a little ahead over flask, though over many different runs the results are practically identical. 

This is more an expected result, as once again the main bottleneck is the database and not the web framework. 

These results show that there is no significant difference between the django implementations. The original discrepancies are easily explained by the the connection pooling.

The only thing that doesn't make sense to me is why django is slower than the flask implementation all else being equal. 

### Digging even deeper
My immediate thought was skill issues, I don't have a lot of experiences with django and it's possible I made a mistake that lead to performance degradation. Checking the code and the orm generated SQL query, nothing really stood out. Then I confirmed that the database driver we're using is indeed psycopg 3 so no differences there.

Finally, I decided to give a closer look at the connection pool used in django, and that's where it all clicked. django uses [psycopg_pool](https://www.psycopg.org/psycopg3/docs/basic/pool.html) an implementation provided directly by [psycopg](https://www.psycopg.org/). Where as sqlalchemy has its own implementation. The differences are actually significant, 

- sqlalchemy has a lazy pool implementation, which means that connections are created or destroyed only when the pool is being accessed.
- ⁠On the other hand, psycopg-pool actively maintains the pool and processes tasks using background thread workers.

So one more time, I modified my fastapi and flask implementation to use the same connection pool as Django:
```py

@app.get("/quote/")
async def quote() -> PlainTextResponse:
    async with pool.connection() as conn:
        async with conn.cursor() as cur:

            await cur.execute(
                "SELECT q.id, q.quote_text, a.name FROM quotes_quote q "
                "JOIN quotes_author a ON q.author_id = a.id "
                "ORDER BY RANDOM() LIMIT 1"
            )
            row = await cur.fetchone()
            return PlainTextResponse(f"{row[1]}\n\n--{row[2]}")

```

And I got the following results:


Server type|workers|threads|RPS|Latency avg|Latency max|Median
---|---|---|---|---|---|---
FastAPI|1|-|687.26|92.95ms|326.49ms|110.29ms
FastAPI (psycopg_pool)|1|-|526.21|121.49ms|248.73ms|128.85ms
Flask|3|5|682.73|93.66ms|239.67ms|104.34ms
Flask (psycopg_pool)|3|5|532.82|119.95ms|289.91ms|172.75ms
Django|3|5|411.91|155.17ms|299.19ms|169.44ms
Django Async|1|-|406.90|152.79ms|279.93ms|160.32ms

So it's quite clear here that [psycopg_pool](https://www.psycopg.org/psycopg3/docs/basic/pool.html) is the root cause in this particularly scenario.

There is still some differences between django and the other frameworks, but I think that's likely because we're using raw sql queries over django's orm. 


### Conclusion
If you were hoping for an answer of "just use async" or "don't use async", life is not so simple. The lesson I hope you take is that there are nuances when it comes to this topic. 

Time and time again there's been posts on async performances, and we often see misleading benchmarks or analysis. 

Let's sum up my own thoughts:

- Asyncio's advantage has never been speed in particular, but the cost. We can achieve the same performance with no added threads or processes.
- Even then you might consider using django or other sync frameworks if you're more familiar with them or if they provide something you don't get elsewhere, e.g. django's rich middleware plugin system.
- Benchmarks cannot be taken at face value, try to reproduce it for your requirements (LLMs are pretty good at this).
- Lastly when you get a result from a benchmark that doesn't match expectation, it's prudent that you investigate where the discrepancy comes from.

### Future Steps
This whole exercise, as exhausting as it is, barely scratches the itch. There's quite a few unexplored questions:

- What is the advantage of [`psycopg_pool`](https://www.psycopg.org/psycopg3/docs/basic/pool.html)'s implementation, are there cases where it performs better?
- Are there any configuration or ticks I'm missing to speed up [Django](https://www.djangoproject.com/)?
- There's not a lot of analysis on async django as it uses threads to run database queries and middlewares, but I think it's worth looking at which situations where async django wins.
- Is there a general rule for thread/process numbers that maximise the performances and what are the trade offs of having more threads.

In the near future I intend to put together a more comprehensive comparison of different concurrency models and explore these questions.

The best I could do right now is to provide the full [source code](https://github.com/Jamie-Chang/web-async-benchmark) and hopefully encourage people to make their own measurements and share their results. In case there's been any mistakes the feedback is very much welcome there.
