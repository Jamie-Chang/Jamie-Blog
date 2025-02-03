Title: Sqlalchemy Footgun: Discarding the statement
Date: 2025-02-02
Category: Blog

This one has frustrated me for a while. 

It starts off with a REST API route. For example in fastAPI

```py
@app.get("/")
def search_users(session: Session) -> list[User]:
    """Finds users optionally filter by user_id"""
    statement = select(User).order_by(User.name)
    return session.execute(statement).scalars().all()
```

Then we get asked to filter by a `User.name` or something else:

```py
@app.get("/")
def search_users(session: Session, name: str | None) -> list[User]:
    """Finds users optionally filter by user_id"""
    statement = select(User).order_by(User.name)
    if name:
        statement.where(User.name == name)
    return session.execute(statement).scalars().all()
```

And we're done! Right? 

Nope, `statement.where` creates a new Select object, but since that object is not assigned to anything it is never used. 
The `statement` we execute with is the original `select(User).order_by(User.name)`. We then get all the users in the database 
as opposed to the specific user we were looking for.

## Solutions
This has happened to me enough times that I wanted to do something about it.

### Testing
Well it goes without saying, go and test your endpoint with all the possible parameters. Whilst this should always be done, it takes 
a longer time for problems to be discovered.

For this problem I will focus on other approaches that may allow it to be caught sooner. 

### New Type Annotation
To me this issue should be caught by static type checkers, we want to catch this issue as soon as possible so something that works as an LSP is ideal.

This is a potential type annotation for it:
```py
class Select:
    def where(self, *value) -> NoDiscard[Self]:
        ...
```

References:

- [Python.org discussion](https://discuss.python.org/t/proposal-typing-no-discard-a-decorator-to-indicate-that-the-return-value-should-not-be-discarded/33138) 
- [MyPy discussion](https://github.com/python/mypy/issues/6936)

I think this definitely should be added and still hope that this gets turned into a PEP and added to Python. But unfortunately there's just too many directions
this ideas is pulled in, so we will need to wait and see.

### Linting and Type Checking
I found out that [Pyright](https://github.com/microsoft/pyright) actually offers the following configuration:
> reportUnusedCallResult [boolean or string, optional]: Generate or suppress diagnostics for call statements whose return value is not used in any way and is not None. The default value for this setting is "none".

This only checks for functions whose return value is not `None` so at least it won't flag functions like `print`. 

But there are still 
functions that this will unnecessarily flag up. For example, 

```py
async with asyncio.TaskGroup() as tg:
    tg.create_task(some_task(...))
```
`create_task` returns a `Future` but it's not necessary to use it. The over all noise on my code base was just too much here.
But it's worth a try to see how many false positive you get on your code base.


### The Builder Pattern
Looking at the problem from a different angle.

The issue is `Select.where` creates a brand new `Select` object so has no side effects (pure function), 
and I'm introducing bugs by confusing a pure function with one that has side effects.

So a simple fix is to make all operations impure.

We can wrap the `Select` object using the following class:

```py
class Builder:
    def __init__(self, statement: Select) -> None:
        self.statement = statement

    def where(self, *conditions) -> None:
        self.statement = self.statement.where(*conditions)
```
Incidentally this is known as the builder pattern, methods on an object build up a mutable internal state and 
finally when we're done building we extract the internal state.


### The Operator Approach
The approach I went with is a bit more "interesting" but it's perhaps not as pythonic.

Consider the corrected code from my example: 
```py
statement = statement.where(User.name == name)
``` 
One big reason I forget the `statement = ` is that it's just a big more verbose. If only there was a short hand to call `.where` and immediately assign it.

That's exactly how inplace operator in python work:

Instead of 
```python
value = value + 1
```

We write:
```python
value += 1
```

Almost all the operators can be overridden at an class level. Libraries like [pipe](https://pypi.org/project/pipe/)
overrides `__or__` (more precisely `__ror__`) and let's users use `|` to chain functions.

So we can wrap any callable that takes a single argument with:
Code sample in [pyright playground](https://pyright-play.net/?code=GYJw9gtgBAJghgFzgYwDZwM4YKYagSwgAcwQFZEV0sAoUSKBATyPwDsBzA408gYTip0AI1TYaNAALwkaTBhpysUAAr4i2OKOwBtACoAaKACUAugC4aUa1ABE9gOog4RKHCjJBIsW7Yw3QmAA7gTkCGBQwtgEbABuYKgA1tj%2BQfgIABZQAAYAPtkAdFY29rYSNlDAbOZQAkJaYjr6pkZm5TYw2MBQAPo9nkJ9ABQ4qMBGsYIArtg1egCUUAC0AHwmlhUVpXWojBnRQc5EGv7AU2zICPhgbAWlxZtQINgIUyBsUKPABVVDk6gzebtaydbp9cAgYZfCbTWZQBbLNbGDaPOz2ACqOCgrA0qHY0XCHi8ewORxOlXOl2ut3uqKeLzeHy%2BPzYf1hQIkoN6GAAjlM4M8hgAPGrsBCLVYxBAo6zPV7vKBCqAAKkVEhovP5z38AF5VOpNNohj1NQLsED-jMoHqAEw0S3RXKfPlmmD22FQXJ603aiRAA)

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class Pipeable[T, R]:
    """Wrap a callable and allow it to be involked with `|`.
    """

    fn: Callable[[T], R]

    def __call__(self, value: T) -> R:
        """Call the wrapped function."""
        return self.fn(value)

    def __ror__(self, value: T) -> R:
        """Use pipeline to call the wrapped function."""
        return self.fn(value)


def _square(x: int) -> int:
    return x * x


squared = Pipeable(_square)
value = 2
value | squared
value |= squared
```

This allows us to use the shorthand `|=` with any function, making our code more concise and less error prone.

The added benefit with Pyright is that it treats use `|` operator as an creating an expression, so `value | squared` will produce the "Expression value is unused" error. 
No analogous feature exist in MyPy yet but this is great for people who use Pyright.

Putting this together, we can do something like this:

```py
statement = select(MyModel).order_by(MyModel.created_at)
if name:
    statement |= where(MyModel.name == name)

if reverse:
    statement |= order_by(MyModel.created_at.desc())
```

If you think this is useful or just want to try it for fun, I've built a package called [sqlalchemy-builder](https://github.com/Jamie-Chang/sqlalchemy-builder).

If you don't want to install it, the recently released [pydantic.run](https://pydantic.run/store/a075fad4d27ad9d0) is perfect for playing around with the code in your browser!


## What have we learned?
From this experience, I'm seeing that there are some gaps in typing in Python, and it's unclear what changes are prioritise and what changes are ultimately
left out of the language. I think a proper typing support is still preferred compared to the other methods which are more like workarounds.

The discovery of operator overloading and Pyright as way to detect unused return value is a pleasant surprise and I hope to find other
use cases for it. Using more syntax in Python might become controversial but I think it depends on the use case, for example, [pathlib](https://docs.python.org/3/library/pathlib.html) makes
heavy use of `/` operator.

I'd love to hear feedback on `sqlalchemy-builder`, this solves a real problem for me, but I'm not sure if other people run into this issue.
