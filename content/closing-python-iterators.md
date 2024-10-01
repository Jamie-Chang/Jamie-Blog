Title: Closing Python Iterators
Date: 2024-09-28
Category: Blog

I recently came across a [video](https://youtu.be/N56Jrqc7SBk?si=6WmYrq8C4E-gwn4_) by mcoding about Python iterators' unfortunate closing behaviour.

To sum up the video, when an iterator is interrupted we intuitively we would expect the exit of a context manager or a finally block to be called but that's not **necessarily** the case. For example,
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

And as the video pointed out, this won't work if we keep any reference to the iterator. Nor is it necessarily the case for all Python implementation that we always garbage collect immediately. 

Additionally the asyncio version of this is even more problematic as we won't be able to run the async `aclose` during garbage collection.

The [PEP-533](https://peps.python.org/pep-0533/) proposal for fixing this ultimately cannot be accepted due to backwards compatibility concerns.

### Calling `.close()`
The best way to resolve this issue is to call `.close()` explicitly. Or with `contextlib.closing`. This is guaranteed to work and be correct but places the responsibility on the caller of the generator to close the iterator. 

The issue is, whether the iterator needs to be closed is implementation specific. So the caller either has to step into the implementation or verbosely call close for every generator.

### Solutions
I believe there to be two correct ways to solve this issue, in lew of the PEP solution. Both method change the callee's signature to avoid any ambiguity.

#### Option 1
Never use `finally` or `with` inside generator functions.

This sounds pretty terrible, but in practice is probably not that big of an issue. The reasons to use `with`/`finally` tend to be for closing resources, so we can just shift where we define and close the resource:

```python
with create_db() as db:
    for row in query(db);
        ...
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
from contextlib import closing
from typing import Callable, ContextManager, Generator 


def contextgenerator[T: Generator, **P](gen: Callable[P, T]) -> Callable[P, ContextManager[T]]:
    """
    Wraps generators in a context manager.
    
    This ensures generators are closed properly.
    """
    @wraps(gen)
    def closing_gen(*args: P.args, **kwargs: P.kwargs) -> ContextManager[T]:
        return closing(gen(*args, **kwargs))
    
    return closing_gen
```

Then we just need to wrap the generator in a `@contextgenerator` like this:
```python
@contextgenerator
def messages():
    ...
```

And called this way:
```python
with messages() as message_iterator:
    for message in message_iterator:
        ...
```

Here we rely on the decorator to communicate the change in signature to the end user.

But the type checker will also pick up the fact that `messages` now returns a context manager as opposed to an iterator.

The benefit here is that there is no way to start the generator without entering the context manager, thus ensuring that the generator closes.

The caller can choose to use this when closing behaviour is required, or use a plain generator when not.

For asyncio we have a similar solution:
```python

from functools import wraps
from contextlib import aclosing
from typing import Callable, AsyncContextManager, AsyncGenerator


def asynccontextgenerator[T: AsyncGenerator, **P](gen: Callable[P, T]) -> Callable[P, AsyncContextManager[T]]:
    """
    Wraps generators in a context manager.
    
    This ensures generators are closed properly.
    """
    @wraps(gen)
    def closing_gen(*args: P.args, **kwargs: P.kwargs) -> AsyncContextManager[T]:
        return aclosing(gen(*args, **kwargs))
    
    return closing_gen
```

### Conclusion

This issue is something that would catch me out on occasion, thanks to mcoding's video I finally have a good understanding on what's going on.

I think in leu of a more sophisticated fix like PEP-533, we could use a combination of the two approaches I've given to mitigate the issue. This can be done by library authors or developers when defining the generators and doesn't rely on the caller to close them explicitly.
