Title: Closing Python Iterators
Date: 2024-10-05
Category: Blog

I recently came across a video by mcoding about Python iterators' unfortunate closing behaviour.

To sum up the video, when an iterator is interrupted we intuitively we would expect the exit of a context manager or a finally block to be called but that's not **nessesarily** the case. For example,
```python
def generator_with_close():
    print("Starting")
    try:
        yield
    finally:
        print("Closing")
```

if we then have the following code interrupt the generator:
```python
for _ in generator_with_close():
    break
```

We actually get what we expected:
```
Starting
Closing
```
But this is due to Python's garbage collection implicitly calling the close method on the generator.

And as the video pointed out, this won't work if we keep any reference to the iterator. Nor is it necessarily the case for all Python implenmentation that we always garbage collect immediately.

The PEP proposal for fixing this ultimately cannot be accepted due to backwards compatibility concerns.

### Calling `.close()`
The best way to resolve this issue is to call `.close()` explicitly. Or with `contextlib.closing`. This is guarenteed to work and be correct but places the responsibility on the caller of the generator to close the iterator. 

The issue is, whether the iterator needs to be closed is implementation specific. So the caller either has to step into the implementation or verbosely call close for every generator.

### Solutions
I believe there to be two correct ways to solve this issue, in lew of the PEP solution. Both method change the callee's signature to avoid any ambiguity.

#### Option 1
Never use `finally` or `with` inside generator functions.

This sounds pretty terrible, but in practice is probably not that big of an issue. The reasons to use `with`/`finally` tend to be for closing resouces, so we can just shift where we define and close the resource:

```python
with create_db() as db:
    for row in query(db);
        `...
```

Generally, I believe this is a better pattern. We often use dependencies more than once. And this is a case where explicit is better. 

#### Option 2
I will concede that not all cases are covered by Option 1. For example, I have this following snippet of code to fetch messages from a broker:

```python
def messages():
    for message in broker.messages():
        with set_context(id=message.id):
            yield message.json()
```

I can lift `set_context` out of the generator. But it certainly makes more sense to set the id inside as the id is not needed outside.

So I think it's better to have an explicit way to force people to close the generator. 

We can achieve this by converting the generator function to a contextmanager function:


```python
from functools import wraps
from contextlib import contextmanager
from typing import Callable, ContextManager, Generator 


def need_close[T: Generator, **P](gen: Callable[P, T]) -> Callable[P, ContextManager[T]]:
    @wraps(gen)
    def closing_gen(*args: P.args, **kwargs: P.kwargs) -> ContextManager[T]:
        return closing(gen(*args, **kwargs))
    
    return closing_gen
```

Then we just need to wrap the generator in a `@need_close` like this:
```python
@need_close
def messages():
    ...
```

And called this way:
```python
with messages() as m:
    for message in m:
        ...
```

The decorator should communicate the change in signature to the end user. The type checkers will also pick up the fact that `messages` returns a context manager.

Even if we called messages directly in a for-loop we just get back a type error. I think the trade off of having slightly less clear signature is worth it as long as the pattern becomes a bit more wide spread in Python.

The pattern is actually already explicitly implemented in other ways in many libraries.
