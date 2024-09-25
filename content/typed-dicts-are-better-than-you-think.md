Title: TypedDicts are better than you think
Published: 2024-10-01
Category: Blog

`TypedDict` was introduced in [PEP-589](https://peps.python.org/pep-0589/) which landed in Python 3.8.

The primary use case was to create type annotations for dictionaries. For example,

```python
class Movie(TypedDict):
    title: str


movie: Movie = {"title": "Avatar"}
```

I remember thinking at the time that this was pretty neat, but I tend to use `dataclass` or `pydantic` to represent 'record' type data. Instead I use dictionaries more as a collection, so the standard `dict[KT, VT]` annotation is enough.

WI