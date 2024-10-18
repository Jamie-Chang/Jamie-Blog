Title: TypedDicts are better than you think
Date: 2024-10-01
Category: Blog

`TypedDict` was introduced in [PEP-589](https://peps.python.org/pep-0589/) which landed in Python 3.8.

The primary use case was to create type annotations for dictionaries. For example,

```python
class Movie(TypedDict):
    title: str


movie: Movie = {"title": "Avatar"}
```

I remember thinking at the time that this was pretty neat, but I tend to use `dataclass` or `pydantic` to represent 'record' type data. Instead I use dictionaries more as a collection, so the standard `dict[KT, VT]` annotation is enough.

### Non-totality
I revisited typeddicts when I looked at implementing a HTTP patch endpoint.

Let's suppose I have a data structure represented by the following dataclass:

```python
@dataclass
class User:
    id: UUID
    name: str
    subscription: str | None = None
```
Where `subscription = None` means no subscription.

Let's say we want to option to patch name, subscription. You might define the patch body using dataclass:
```python
@dataclass
class PatchUser:
    name: str | None = None
    subscription: str | None = None
```

Here we have a problem, for subscription does `None` mean don't change or remove subscription. 

We can fix this a number of ways, for example, we can take the string `'none'` to mean no subscription instead, or make a new sentinel value called `NoChange` to indicate no changes.

These solutions all feel a little awkward, this is because dataclasses don't have a concept of a field being missing. But this is where dictionaries shine. Dictionaries are not general expected to have all the fields available. We get a `KeyError` if a field is missing and there are convenience methods such as `.get(key, [default])` to fetch a key that is not guaranteed to be present.

This makes `TypedDict` the ideal data structure in this scenario:

```python
class PatchUser(TypedDict, total=False):
    name: str | None
    subscription: str | None = None
```

Since `total` is False here (by default it is set to True), `name` or `subscription` can be absent from the dictionary. Which represents the PATCH operation much better than a `dataclass` or Pydantic model.

Further additions in [PEP-655](https://peps.python.org/pep-0655/) allows us to mark individual fields as `Required` or `NotRequired` which further increases its flexibility.

> If you're wondering about FastAPI support for TypedDict, [Pydantic supports it out of the box](https://docs.pydantic.dev/2.3/usage/types/dicts_mapping/#typeddict). So your TypedDict can be used in a FastAPI endpoint.

### Using `TypedDict` as `**kwargs`
[PEP-692](https://peps.python.org/pep-0692/) introduced the ability to type variadic keyword arguments using `TypedDict`.

So the following two snippets are equivalent.
Without `TypedDict`:
```python
def my_function(*, option1: int, option2: str) -> None:
    ...
```

Using `TypedDict`:
```python
from typing import TypedDict, Unpack


class Options(TypedDict):
    option1: int
    option2: str


def my_function(**options: Unpack[Options]) -> None:
    ...
```

At a glance I can say that the TypedDict option is rather verbose. Though it does become more useful if Options were used in multiple function definitions.

```python
def my_function2(**options) -> None:
    ...


def my_function3(*, other_option: str, **options) -> None:
    ...
```
Where it truely shines is once again with non-totality.

Suppose we have the following scenario, where we want to create a custom version of pytest.fixture, but still pass through some arguments.
```python
def fixture(scope: str = "module", autouse: bool = False):
    return pytest.fixture(scope, autouse)
```

Here to get the typing right I not only have to find the type of each argument but also the default value. It would be better if we use `**kwargs` so we can just avoid passing the arguments through. And to keep type information we just need to use our trusty `TypedDict` once more:


```python
class FixtureOptions(TypedDict, total=False):
    scope: str
    autouse: bool


def fixture(**options: Unpack[FixtureOptions]):
    # Some custom implementations
    ...
    return pytest.fixture(**options)
```

Non-totallity means that we don't have to pass in scope and autouse. We can just have the default.

#### Sentinels
We can achieve similar behaviour with sentinels:
```python
UNSPECIFIED: Any = object()  # Has to be Any type so it could be set as default for other types.

def my_func(option1: bool = UNSPECIFIED, ...) -> ...:
    if option1 is UNSPECIFIED:
        ...
    ...
```

Sentinels work well enough here, but we have to remember to handle them. Additionally type annotations for sentinels can be a bit awkward, here we made `UNSPECIFIED` an `Any` type, but it means that inside the function `option1` is only typed as `bool`. There are options to expose the sentinel type but they may add even more confusion.



### Using `TypedDict` to pass in dependencies
We can do even more with [PEP-692](https://peps.python.org/pep-0692/)! When I first learned about the PEP, I thought it was only about function signature. But reading through it more thoroughly, I discovered that another consequence of the PEP is that type checkers can now check for function invocation when using TypedDicts:

```python

def purge(queue: str, timeout: float) -> ...:
    ...


class Options(TypedDict):
    queue: str
    timeout: float


class WrongOptions(TypedDict):
    queue: str
    timeout: timedelta


options: Options = ...
purge(**options)  # ✅

wrong_options: WrongOptions = ...
purge(**wrong_options)  # ❌
```

This feature is necessary in many situations such as cases where we pass through the kwargs. For example, in the `fixture` example, when we invoke `pytest.fixture(**options)` the type checker will perform proper type checking.

But we can use it in more creative ways.

#### Dependency Injection
Let's consider a situation where we have many resources that share some dependencies. 

```python
class UserClient:
    def __init__(self, db: Engine, user_service: APIClient) -> None:
        ...


class ProjectClient:
    def __init__(self, db: Engine, user_service: APIClient, project_service: APIClient) -> None:
        ...
```

We want a way to create all the dependencies in one place and pass in the dependencies.

Essentially we need something that is the union of all kwargs of the resources. That suddernly sounds a lot like a TypedDict:

```python
class Dependencies(TypedDict):
    db: Engine
    user_service: APIClient
    project_service: APIClient


def create_deps(...) -> Dependencies:
    ...
```

Unfortunately this won't work since `UserClient` can't take `project_service` as a kwarg.

To fix this, we need to rewrite the resources such that we accept arbitrary arguments.

```py
class UserClient:
    def __init__(self, ..., **_) -> None:
        ...
...
```


And then we can do the injection like this:
```python
class ResourceWithMissing:
    def __init__(self, other: Any, **_) -> None:
        ...


def inject(deps: Dependencies):
    UserClient(**deps)  # ✅
    ProjectClient(**deps)  # ✅
    ResourceWithMissing(**deps)  # ❌
    ...


inject(create_deps(...))
```

With the solution complete, we can now rely on the type system to check the dependency injection to see if any arguments are incorrect or missing. 

I will admit that changing resource signature with `**_` is not ideal, but this is a smaller change than most dependency injection frameworks. And we get static type checking which a lot of the frameworks won't support.

### Upcoming Features
[PEP-728](https://peps.python.org/pep-0728/) will allow types of extra items to be defined, and a typed dict to be closed meaning no extra items can be defined.

This new change looks like it'll help us define record types more precisely.

I personally haven't thought of many other use cases for it, but as I've demonstrated above it's always worth reading through the PEP and experimenting with the new change. 

[PEP-705](https://peps.python.org/pep-0705/) might already be out by the time you read this. This will allow for read only items to be specified.

This is primarily intended for situations where different typed dicts intuitively should be compatible but potential mutations (deletions) can create problems.
