Title: Customising Pattern Matching Behaviour
Date: 2024-12-09
Category: Blog

I've been doing [advent of code](https://adventofcode.com/) again this year. There are two Python features I always rely on, iterators and pattern matching. Iterators allow for operations on each of its elements without allocating memory for a collection. Ever since pattern matching was introduced in Python 3.10, it's been particularly useful to unpack nested object structure. 

Unfortunately pattern matching and iterators don't work well together. By definition elements of an iterator do not necessarily exist so there's nothing to match against. Often to pattern match we must first consume the iterator:

```python
match range(10): 
    case [0, *_]:  # Won't match of course
        ...

match list(range(10)):
    case [0, *_]:  # matches now since iterator is consumed
        ...
```

Ideally we'd be able to consume the head elements of the iterator on demand, but this isn't possible directly with iterators.

### Official Support for Custom Matching
In the original PEP, there's mention of the idea of [custom matchers](https://peps.python.org/pep-0622/#custom-matching-protocol). But the idea was ultimately deferred due to the lack of a clear use case. 

### Matching Properties
But it turns out we don't really need any of this. We can match any named attributes of an object even a `property`.

```python
@dataclass
class Repeat:
    value: int
    times: int

    @property
    def repeated(self) -> list[int]:
        return [self.value] * self.times


match Repeat("abcd", 2):
    case Repeat(times=times, repeated=list() as repeated):
        print(f"After {times} times", repeated)
```
This is really useful as we need to dynamically unpack the iterator if we want to match the iterator. We can create an object with the `next` property that evaluates the next elements on demand.

```python
@dataclass
class Node:
    value: int
    it: Iterator[int]

    @classmethod
    def from_iter(cls, it: Iterator[int]) -> Node:
        return cls(next(it), it)

    @property
    def next(self) -> Node:
        return Node(next(self.it), self.it)


match Node.from_iter(iter(range(10))):
    case Node(value=0, next=Node(value=1, _)):
        print("Starts with 0 and 1")
```

There is a bug in the code, consider the following match statement:
```python
match Node.from_iter(iter(range(10))):
    case Node(value=9, next=Node(value=8, _)):
        print("Starts with 9 and 8")
    case Node(value=0, next=Node(value=1, _)):
        print("Starts with 0 and 1")
```

The looks like it'll match in the second `case`, but it can't. This is because the first `case` checks the first 2 elements by consuming them. 

But the fix is actually really simple, we simple use `cached_property` to cache the result of `next`:
```python
@dataclass
class Node:
    value: int
    it: Iterator[int]

    @classmethod
    def from_iter(cls, it: Iterator[int]) -> Node:
        return cls(next(it), it)

    @cached_property
    def next(self) -> Node:
        return Node(next(self.it), self.it)
```

### Matching positionally using `__match_args__`
Matching named attributes can be very verbose. We can specify which arguments to match sequentially by adding  [`__match_args__`](https://peps.python.org/pep-0622/#special-attribute-match-args) to the class:

```python
@dataclass
class Node:
    value: int
    it: Iterator[int]

    __match_args__ = ("value", "next")

    @classmethod
    def from_iter(cls, it: Iterator[int]) -> Node:
        return cls(next(it), it)

    @cached_property
    def next(self) -> Node:
        return Node(next(self.it), self.it)


match Node.from_iter(iter(range(10))):
    case Node(9, Node(value=8, _)):
        print("Starts with 9 and 8")
    case Node(0, Node(value=1, _)):
        print("Starts with 0 and 1")
```

### Matching the end of iteration
Finally we need to consider the `StopIteration` at the end of the iterator.

```python
@dataclass
class Node:
    value: int
    it: Iterator[int]

    __match_args__ = ("value", "next")

    @classmethod
    def from_iter(cls, it: Iterator[int]) -> Node | None:
        try:
            return cls(next(it), it)
        except StopIteration:
            return None

    @cached_property
    def next(self) -> Node | None:
        try:
            return Node(next(self.it), self.it)
        except StopIteration:
            return None


match Node.from_iter(iter(range(1))):
    case Node(9, Node(value=8, _)):
        print("Starts with 9 and 8")
    case Node(0, None):
        print("Starts with 0")
```

### Conclusion
You might still be wondering if this is actually useful. I've published this as [pattern-utils](https://github.com/Jamie-Chang/pattern-utils) so you can try it out. Beware the implementation is slightly different to facilitate matching generator with return, for example,

```python
from pattern_utils import generator as gen


def example_generator():
    yield "some resource"
    return "done"


match gen.matcher(example_generator()):
    case gen.Node(resource, gen.Empty(end_result)):
        print(resource, end_result)
```

I think more important is using the same technique of writing a wrapper with properties to better exploit pattern matching. If I discover other common use cases I will update the library.