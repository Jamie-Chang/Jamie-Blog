Title: Finally trying out Mojo 🔥
Date: 2025-10-02
Category: Blog
Tags: Mojo, Python, Advent of Code

[Mojo](https://www.modular.com/mojo) has been on my radar for a while. In fact, I heard about it on launch back in May 2023. It was touted as a super set of Python with 10000s of times the performance. It's main focus is on AI related workloads with strong built in support of GPU programming.

I've used [Advent of code 2024 day 6](https://adventofcode.com/2024/day/6) as a benchmark for in Python before in [Python 3.14: State of free threading]({filename}/free-threading-3-14.md). So I figured why not solve the same puzzle one more time.

## Implementation
Implementation can be found here:
<style type="text/css">
  .gist-file
  .gist-data {max-height: 500px;}
</style>

<script src="https://gist.github.com/Jamie-Chang/bcc693baebd6bb5ce471791cc57f0f6e.js"></script>

I won't go into all the details instead I'll choose to highlight parts of the code that are interesting.

### Python like syntax
Python's syntax can be divisive, I happened to really like it (I'm biased). Mojo's syntax definitely carries over the spirit of Python's. 

```mojo
@fieldwise_init
struct Grid[T: Movable & ImplicitlyCopyable](Copyable, Movable):
    var data: List[List[T]]

    fn __contains__(self, pos: Pair) -> Bool:
        return 0 <= pos.first < len(self.data) and 0 <= pos.second < len(
            self.data[pos.first]
        )

    fn __getitem__(self, pos: Pair) -> T:
        return self.data[pos.first][pos.second]

    fn __setitem__(mut self, pos: Pair, value: T):
        self.data[pos.first][pos.second] = value
```
The above code is very similar to a dataclass I would create in Python:

```python
@dataclass
class Grid[T]:
    data: list[list[T]]

    def __contains__(self, pos: Pair) -> bool:
        return 0 <= pos.first < len(self.data) and 0 <= pos.second < len(
            self.data[pos.first]
        )

    def __getitem__(self, pos: Pair) -> T:
        return self.data[pos.first][pos.second]

    def __setitem__(self, pos: Pair, value: T) -> None:
        self.data[pos.first][pos.second] = value
```

So far so good, `@fieldwise_init` on a struct in mojo is pretty much equivalent to `@dataclass` in Python. The function syntax is pretty much the same. We need to declare [traits](https://docs.modular.com/mojo/manual/traits/) similar to rust but otherwise this all makes sense. 

### Ergonomics

Whilst the syntax has the look and feel of Python. It's a little sad that a lot of python's conveniences haven't arrived in mojo yet. 

The biggest being the lack of generators. As a Python programmer I'm yielding whenever I can. It's a quick and easy way to define iteration behaviour. Mojo let's you define iterators as structs but it's pretty tough, see how I defined an iterator for how a guard should move:

```mojo
struct Guard(Copyable, Iterable, Iterator, Movable):
    alias Element = Pair
    alias IteratorType[
        iterable_mut: Bool, //, iterable_origin: Origin[iterable_mut]
    ]: Iterator = Self

    var grid: ArcPointer[Grid[Bool]]
    var position: Pair
    var direction: Pair
    var obstacle: Optional[Pair]

    fn __init__(
        out self,
        position: Pair,
        read grid: ArcPointer[Grid[Bool]],
        obstacle: Optional[Pair] = None,
    ):
        self.position = position
        self.grid = grid
        self.direction = Pair(-1, 0)
        self.obstacle = obstacle

    fn __has_next__(self) -> Bool:
        return self.position in self.grid[]

    fn __iter__(ref self) -> Self.IteratorType[__origin_of(self)]:
        return self.copy()

    fn __next__(mut self) -> Pair:
        var old = self.position
        candidate = self.position + self.direction
        while (
            candidate in self.grid[]
            and self.grid[][candidate]
            or (self.obstacle and candidate == self.obstacle.value())
        ):
            self.direction = self.direction.rotate()
            candidate = self.position + self.direction

        self.position = candidate
        return old

```

There is a lot of boiler plate needed for this, though it's not totally unreasonable. Using `yield` in Python would of course be a lot easier. But if we were forced to make a class based iterator then it'll look similar. 

Another feature I missed is pattern matching, but it's hopefully coming soon.

### Traits
[Traits](https://docs.modular.com/mojo/manual/traits/) are used to define some behaviour an object should have in mojo. Traits were made popular in [rust](https://doc.rust-lang.org/book/ch10-02-traits.html). But is basically like an interface in Java but can have default method implementations. In Python this is closest to an abstract base class. 

```mojo
struct Guard(Copyable, Iterable, Iterator, Movable):
    ...
```

Here `Copyable, Iterable, Iterator, Movable` are all built in traits, for example `Iterable` trait is needed so we can use `Guard` in a for loop:

```mojo
for positions in guard:
    ...
```

And traits can be more powerful than Python types because they can be composed using `&`. Where as Python currently doesn't have type intersections. 

As someone who does a lot of type annotations in Python, I like the power traits can give you here. But there are some sharp corners:

- Some things are still not representable: `Tuple` in mojo is not `Hashable` this is because whether it's hashable or not depends on the child types, and there is no way in mojo to represent that.
- There are a lot of traits to remember, it can get old writing `Copyable` and `Movable` over and over again. That's not to mention we have `Implicit` versions of some traits too.


### Memory Ownership
Mojo is a statically typed compiled language, it has an memory ownership model similar to rust and zig. 

Essentially a value is associated with the scope it's defined in, you can transfer this ownership by using ^ either when you're passing arguments to a function and therefore new scope, or when you are returning from a scope.

```mojo
fn parse(path: Path) raises -> Tuple[Grid[Bool], Pair]:
    matrix: List[List[Bool]] = []  # list defined here owned by the current scope
    ...
    return Grid(matrix^), start.value()  # passing the ownership to Grid struct
```

This takes some getting used to, but it does often boil down adding `^` when the compiler yells at you. 

### Lifetimes
The ownership model necessitates the parametrisation of lifetimes, when a value is passed to a struct the compiler needs to know how long the variable is alive for and adjust the struct's lifetime accordingly. 

This is done in mojo via [origin](https://docs.modular.com/mojo/manual/values/lifetimes/)

An example when defining an iterator:

```mojo
struct ResultSetIter[mut: Bool, //, origin: Origin[mut]](
    Copyable, Iterator, Movable
):
    alias Element = Bool

    var result_set: Pointer[ResultSet, origin]
```

Here we parameterise the origin to match the origin for the pointer. 

In my little example, the lifetimes have been the most challenging. I'm aware that lifetimes are also one of the nastiest parts of rust as well. In my case, having very few examples of lifetimes being used with structs available to me really made it more difficult. 

In one case, I gave up and just used an ARC (Atomic Reference Counting) Pointer. Which allows me to manage the value without lifetime, at the cost of unnecessary memory indirection. 

### Concurrency

Mojo doesn't currently have a lot of utilities for concurrency, it doesn't let you create `threads` directly.

I've found [parallelize](https://docs.modular.com/mojo/stdlib/algorithm/functional/parallelize/) which is similar to a thread pool. 


```mojo
    var inputs = [elem for elem in positions]
    var result_set = ResultSet(len(inputs))

    @parameter
    fn worker(row: Int):
        var guard = Guard(start, grid, inputs[row])
        loop = has_loop(guard)
        result_set[row] = loop

    parallelize[worker](len(inputs))
```

To collect the results I've implemented a wrapper around some memory. This allows each item to be stored in a separate piece of memory. Avoiding the need for thread safe data structures that mojo currently doesn't have.

```mojo
struct ResultSet(Copyable, Movable):
    var data: UnsafePointer[Bool]
    var size: Int

    fn __init__(out self, size: Int):
        self.data = UnsafePointer[Bool].alloc(size)
        self.size = size

    fn __copyinit__(out self, existing: Self):
        self.data = UnsafePointer[Bool].alloc(existing.size)
        self.size = existing.size
        memcpy(dest=self.data, src=existing.data, count=existing.size)

    fn __del__(deinit self):
        self.data.free()

    fn __setitem__(self, index: Int, value: Bool):
        (self.data + index)[] = value

    fn __getitem__(self, index: Int) -> Bool:
        return (self.data + index)[]
```

This has been a lot more effort due to the lack of general support for threading. Ideally there's a way to collect the result in shorter batches, and an API that's similar to Python's [`ThreadPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#threadpoolexecutor). It looks like there might be async support coming soon, so there is something to look forward to!

## Performance
I've measured the following performance:

Single core: 1.037 s
Multi core: 0.273 s

This is around 3x the performance of my best python implementations. It'll be interesting to see how this would compare with other system programming languages like rust. 

One of mojo's big features is its [interoperability](https://docs.modular.com/mojo/manual/python/) with Python. So there's a lot of opportunity to speed up your slow python code with mojo and this is before we even look at SIMD and GPUs.

## My feeling around Mojo

Some people have voiced concerns over the fact that mojo is backed by VCs and is mostly closed sourced. 

Personally, I don't have any problems with this, we can't be too idealistic here. Software like [`uv`](https://docs.astral.sh/uv/) might not exist without companies or investors funding it. And there is [commitment](https://docs.modular.com/mojo/faq/#will-mojo-be-open-sourced) to open source it eventually.

I think what mojo is trying to achieve is certainly interesting. Whilst it started out as a "Python Superset" it's not turned into a mix of Zig, Rust + Python syntax + inbuilt GPU/SIMD support. 

If you think that's a lot, you'd be right! You may have noticed as we walk through mojo's features, I find myself comparing to rust most of the time and not Python. People who don't know an existing systems programming language may find it too hard. People with experience in these languages may decide to stay in those languages.

But on the flip side, if they can achieve this then we may have a really powerful language. 

Lastly I want to comment on the documentation around mojo. Though the tutorial is fairly good, if you veer just off the beaten track, then you get very very basic [docs](https://docs.modular.com/mojo/stdlib/pathlib/path/Path) with no examples. For this I had to reference the source code for the [standard library](https://github.com/modular/modular/blob/main/mojo/stdlib/README.md), which is luckily the part of mojo that is open source. 


## Next steps
There are 2 aspects of mojo I've yet to try:

- GPU programming: I just don't have the hardware to take advantage of it
- Python interop: As I mentioned earlier, this could be a big win for mojo. Certainly something I will put some time to in the future.
