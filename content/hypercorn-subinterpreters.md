Title: You can (kind of) run Django in subinterpreters
Date: 2026-04-26
Category: Blog
Tags: Python, Subinterpreters

In November of 2023 cpython core developer Anthony Shaw wrote about sub-interpreters in his post: [Running Python Parallel Applications with Sub Interpreters](https://tonybaloney.github.io/posts/sub-interpreter-web-workers.html), where he previewed the ongoing development for subinterpreters and highlighted his attempt to run web applications in a subinterpreter.

Anthony concluded his post having adapted [hypercorn](https://github.com/pgjones/hypercorn) to run with subinterpreters and managing to run FastAPI and Flask applications. He notably couldn't run Django due to the standard library's own `datetime` module. 

It has now been 2 and half years and a lot has changed. Both [concurrent.interpreters](https://docs.python.org/3/library/concurrent.interpreters.html) and [free-threading](https://docs.python.org/3/howto/free-threading-python.html) have seen official releases, with subinterpreters especially seeing a lot of changes with respect to the Python interface. 

## Continuing the Work
So I decided to give it a try myself and I started from scratch given all the changes in Python.

I've not used hypercorn prior to this but it's very similar to uvicorn, which I'm familiar with. Hypercorn runs an asyncio worker in multiple processes, it should in be a straightforward swap from processes to interpreters. 

And it did actually turn out pretty simple. In particular, I was able to avoid a lot of the headaches Anthony had as there's been substantial quality of life changes in the interface. It also helps that I've had an (unhealthy) obsession with subinterpreters lately. 


### Sharing Sockets
Hypercorn shares the same set of sockets between all the worker processes. The processes will all consume this set of sockets thereby distributing requests. Socket's are shareable by design between a parent process and any sub-processes, but that's not the case for sub-interpreters.

In my previous [blog post](https://blog.changs.co.uk/i-was-wrong-about-subinterpreters.html) I came to realise that we can just rely on pickle to share objects between interpreters. We will utilise this mechanism here.

Python lets you to customise what state is passed to the pickle (de)serialiser via [`__getstate__`](https://docs.python.org/3/library/pickle.html#object.__getstate__) and [`__setstate__`](https://docs.python.org/3/library/pickle.html#object.__setstate__). So we just need to decern what data is needed to be shared, and how we reconstruct the object.


```py
class Sockets:
    secure_sockets: list[socket.socket]
    insecure_sockets: list[socket.socket]
    quic_sockets: list[socket.socket]

    def __getstate__(self):
        """Prepares the socket for transport."""
        return {
            "secure_sockets": [
                {
                    "fd": sock.fileno(),
                    "family": sock.family,
                    "type": sock.type,
                    "proto": sock.proto,
                }
                for sock in self.secure_sockets
            ],
            "insecure_sockets": [
                {
                    "fd": sock.fileno(),
                    "family": sock.family,
                    "type": sock.type,
                    "proto": sock.proto,
                }
                for sock in self.insecure_sockets
            ],
            "quic_sockets": [
                {
                    "fd": sock.fileno(),
                    "family": sock.family,
                    "type": sock.type,
                    "proto": sock.proto,
                }
                for sock in self.quic_sockets
            ],
        }

    def __setstate__(self, state):
        """Reconstructs the socket in the new interpreter."""
        for key, socks in state.items():
            setattr(
                self,
                key,
                [
                    socket.fromfd(raw["fd"], raw["family"], raw["type"], raw["proto"])
                    for raw in socks
                ],
            )
```

Simply, this extracts the enough metadata of the socket (e.g. file descriptor number) to pass to the sub-interpreter. The sub-interpreter then calls [socket.fromfd](https://docs.python.org/3/library/socket.html#socket.fromfd) to rebuild the socket in the interpreter. 

This is pretty clean, `Sockets` is an existing class in hypercorn, by making it pickleable we've avoided having to change the interface.

### Calling the Sub-Interpreter
When I first started playing around with interpreters I assumed we had to use the [exec](https://docs.python.org/3/library/concurrent.interpreters.html#concurrent.interpreters.Interpreter.exec) method, with queues for passing data. 

But over an embarrassingly lengthy period of time, I finally [figured out]({filename}/subinterpreters-retrospective.md) that the higher level function works just as well, or maybe even better. For this I used [Interpreter.call](https://docs.python.org/3/library/concurrent.interpreters.html#concurrent.interpreters.Interpreter.call) wrapped inside a thread:

```py

def run_interp[**P](fn: Callable[P, Any], *args: P.args, **kwargs: P.kwargs) -> None:
    interp = interpreters.create()
    try:
        interp.call(fn, *args, **kwargs)
    finally:
        interp.close()


thread = Thread(
    target=run_interp, args=(worker_func, config, sockets, shutdown_indicator)
)
thread.start()
```

It's also possible to use [Interpreter.call_in_thread](https://docs.python.org/3/library/concurrent.interpreters.html#concurrent.interpreters.Interpreter.call_in_thread), but I preferred having the lifetime of the interpreter entirely managed within the thread.

### Stopping the Interpreters
Finally we need to consider how the main interpreter can communicate with the worker interpreters to shutdown gracefully.

Let's first look at how this is done with processes...

The main processes passes an [Event](https://docs.python.org/3/library/multiprocessing.html#multiprocessing.Event) as an indicator for the subprocess to shutdown. 

The subprocess periodically checks to see if the event is set:

```py
async def check_multiprocess_shutdown_event(
    shutdown_event: EventType, sleep: Callable[[float], Awaitable[Any]]
) -> None:
    while True:
        if shutdown_event.is_set():
            return
        await sleep(0.1)

```

It's important to note that whilst event is most commonly [wait](https://docs.python.org/3/library/threading.html#threading.Event.wait)ed on to synchronise the timing in sub-processes. In this case, the event is just used to conveniently store some global state.

The coroutine runs in the background of each worker polling the state with 100ms intervals.  

There is no equivalent to `Event` for interpreters, synchronisation is achieved by waiting on Queues. Since we only need a way to share state, there is a very simple and powerful interpreter mechanism for it: sharing the [memoryview](https://docs.python.org/3/library/stdtypes.html#memoryview). 

Where most objects are shared by copying (pickled). A `memoryview` object is shared by 'reference', which means changes to it are reflected in a different interpreter.  

We will start by creating a `memoryview` wrapping an array of 1 byte. we'll use the value to indicate whether a shutdown is indicated or not. 0 by default to mean no shutdown and 1 to mean shutdown. 


```py
shutdown_indicator = memoryview(array("B", [0]))


async def check_multiprocess_shutdown_event(
    shutdown_indicator: memoryview[bytes], sleep: Callable[[float], Awaitable[Any]]
) -> None:
    while True:
        if shutdown_indicator[0]:
            return
        await sleep(0.1)
```

As shown, the change in code is minimal. The performance is  more likely to be better than events, as the state sharing mechanism is simpler. 

## Testing this out
All the code can be found at [hypercorn-subinterpreters](https://github.com/Jamie-Chang/hypercorn-subinterpreters), I plan on tidying this up more before publishing to pypi. 

```shell
uv add "git+https://github.com/Jamie-Chang/hypercorn-subinterpreters"
```

You can test this out with a simple ASGI[*]({filename}/webasgi.md) webserver, for example:


```py
async def app(scope, receive, send):
    if scope["type"] != "http":
        raise Exception("Only the HTTP protocol is supported")

    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [
            (b'content-type', b'text/plain'),
            (b'content-length', b'5'),
        ],
    })
    await send({
        'type': 'http.response.body',
        'body': b'hello',
    })
```
and then running this with:

```shell
hypercorn main:app --workers 10
```

After a http to localhost, we can see the expected response of 'hello'. 

Compared to the standard version, subinterpreters start up much faster and also avoids issues like zombie processes. But it's not without flaws.

## The Compatibility Landmine
The web app above is as simple as it gets, you can even run it on the [web](https://jamie-chang.github.io/webasgi/?auto=true#code/eyJtYWluLnB5IjoiYXN5bmMgZGVmIGFwcChzY29wZSwgcmVjZWl2ZSwgc2VuZCk6XG4gICAgaWYgc2NvcGVbXCJ0eXBlXCJdICE9IFwiaHR0cFwiOlxuICAgICAgICByYWlzZSBFeGNlcHRpb24oXCJPbmx5IHRoZSBIVFRQIHByb3RvY29sIGlzIHN1cHBvcnRlZFwiKVxuXG4gICAgYXdhaXQgc2VuZCh7XG4gICAgICAgICd0eXBlJzogJ2h0dHAucmVzcG9uc2Uuc3RhcnQnLFxuICAgICAgICAnc3RhdHVzJzogMjAwLFxuICAgICAgICAnaGVhZGVycyc6IFtcbiAgICAgICAgICAgIChiJ2NvbnRlbnQtdHlwZScsIGIndGV4dC9wbGFpbicpLFxuICAgICAgICAgICAgKGInY29udGVudC1sZW5ndGgnLCBiJzUnKSxcbiAgICAgICAgXSxcbiAgICB9KVxuICAgIGF3YWl0IHNlbmQoe1xuICAgICAgICAndHlwZSc6ICdodHRwLnJlc3BvbnNlLmJvZHknLFxuICAgICAgICAnYm9keSc6IGInaGVsbG8nLFxuICAgIH0pIn0). 

I want to instead look at how well hypercorn-subinterpreters will run a web application framework including database drivers.

Luckily, I recently went through the trouble of writing a bunch of web apps using flask, fastapi and django to run some [benchmarks]({filename}/asyncio-benchmarking.md). The apps connect to a local postgres, so they are representative of a real web app.

So here's what worked and what didn't.


### FastAPI
FastAPI is hands down the most popular web framework and is unfortunately the biggest disappointment.

```shell
ImportError: module pydantic_core._pydantic_core does not support loading in subinterpreters
```

The problem is with pydantic. When Anthony tested his version, Pydantic v1 was written in pure Python and Pydantic 2 wasn't out yet. Now pydantic v2 is written in rust using PyO3 and [PyO3 does not currently support subinterpreters ](https://github.com/PyO3/pyo3/issues/3451). PyO3 provides the rust to python bindings, and is used for almost all rust based libraries.

This is a shame, rust-base libraries are only getting more popular in Python, so hopefully this will be resolved soon.

### Django
Django is a much better story. Django itself is mostly pure Python, so it works well running as an asgi application. 

I wasn't expecting it to work with [postgres (psycopg)](https://www.psycopg.org/) as it contains native code. However I was pleasantly surprised to find it working. 

There are unfortunately still caveats here, psycopg won't work when used as a [connection pool](https://www.psycopg.org/psycopg3/docs/advanced/pool.html): 

```shell
RuntimeError: daemon threads are disabled in this (sub)interpreter
```

In my prior benchmarking post, I discovered that psycopg pools actively manages connections using threads. The threads are daemon threads, meaning they exit when the process exits. The sub-interpreters have a lifetime smaller than processes lifetime, so they cannot create and own daemon threads. 

### Flask with Sqlalchemy
Like Django, Flask also runs fine, and with flask we have more flexibility of database libraries. [sqlalchemy](https://www.sqlalchemy.org/) is the natural choice here. And unlike psycopg-pool, it supports pooling without background threads.

```py
from asgiref.wsgi import WsgiToAsgi
from flask import Flask
from sqlalchemy import create_engine

app = Flask(__name__)

# Flask is wsgi, need to be wrapped
asgi_app = WsgiToAsgi(app)

# sqlalchemy engine is Pooled by default
engine = create_engine(os.getenv("DSN"))

...
```

### Other Compatibility Issues
Outside of web applications there are issues in a lot of native code libraries.

#### Data Science
One might imagine a good use case for interpreters are data science applications. They tend to be CPU intensive and a lot of exiting code uses multiprocessing for parallelism. Interpreters should be a natural replacement, however most of the core libraries don't support interpreters. These libraries include 

- pyarrow
- pandas
- numpy
- polars

In contrast most of these libraries have prioritised on free-threading compatibility. You can see their progress in the [tracking page](https://py-free-threading.github.io/tracking/). 

My understanding is that whilst the work to support free-threading and sub-interpreters are not the same, they share some similarities. They tend to involve some reorganisation for native memory management. I believe that interpreters support will come given more time, but I'm not too familiar with the development process for this. 

#### Serialisation
As mentioned before, Pydantic doesn't currently work, but it's not the only serialisation library with this issue. I also noticed problems with [msgspec](https://github.com/jcrist/msgspec/issues/1026). Though [protobuf](https://protobuf.dev/getting-started/pythontutorial/) provided some much needed good news! In my testing it worked out of the box.


#### Pytest
When 3.14 initially came out, I wanted to experiment with running tests in an isolated interpreter. This would not only allow for better parallelism but also shield against unwanted side effects. 

Unfortunately this is also not [supported](https://github.com/pytest-dev/pytest/issues/13809), but there seem to be some interest in this at least. 

## Closing Words
I've been a big proponent of sub-interpreters since its inception, I do believe it has a place in Python even with better support for free-threading across the wider industry.

The incompatibilities were a little surprising, I've always viewed sub-interpreters and multi-processing as analogs of each other. So it's easy to assume things work the same. However, I've come to understand that changing concurrency models breaks some fundamental assumptions about Python and will require non-trivial changes.

I think it's a good idea for myself and others who are interested to get our hands dirty. It's clearly a difficult job to maintain these native libraries in Python so it'll be good for more people to understand and maybe even aid in their development.  