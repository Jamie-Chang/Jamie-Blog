Title: How free are threads in Python now?
Date: 2024-12-31
Category: Blog

Free-threaded Python ([PEP-703](https://peps.python.org/pep-0703/)) was released in October 2024. 
It enables true multi-threaded execution without the restriction of the [GIL](https://docs.python.org/3/glossary.html#term-global-interpreter-lock).

I previously covered this in [Free Threaded Python With Asyncio](./free-threaded-python-with-asyncio.html), the example used there was selected because it very clearly demonstrates the performance increase, or moreover, the threads "running free".

But that's not really code that solves a problem. It's simply repeating the same calculation. As I've been having a lot of fun with advent of code this December, it provided a very good testbed for free threading. 



## AOC Day 6 
> Spoilers for day 6 2024 ahead! full source code [here](https://github.com/Jamie-Chang/advent2024-python)

[Day 6](https://adventofcode.com/2024/day/6) was a good example of a problem that can be solved using parallelisation.

Part 1 involved tracing the path of a guard through a map. Part 2 then asks for locations where we could place a single obstacle and cause the guard to form a loop. 

Here's my full solution:

```python
from __future__ import annotations

from contextlib import contextmanager
from dataclasses import dataclass
from itertools import pairwise
from pathlib import Path
from typing import Hashable, Iterable, Iterator, Self, assert_never


type Pair = tuple[int, int]


up = 0
right = 1
down = 2
left = 3


def turn(direction: int) -> int:
    return (direction + 1) % 4


@dataclass(slots=True)
class Ranges:
    """Traversable location in the map."""
    rows: range
    cols: range

    def __getitem__(self, key: tuple[slice, int] | tuple[int, slice]) -> Iterator[Pair]:
        match key:
            case int() as r, slice() as cols:
                return ((r, c) for c in self.cols[cols])
            case slice() as rows, int() as c:
                return ((r, c) for r in self.rows[rows])
            case _ as other:
                assert_never(other)

    def walk(self, start: Pair, direction: int) -> Iterator[Pair]:
        match direction:
            case 0:
                return ((row, start[1]) for row in self.rows[start[0] :: -1])
            case 2:
                return ((row, start[1]) for row in self.rows[start[0] :: 1])
            case 3:
                return ((start[0], col) for col in self.cols[start[1] :: -1])
            case 1:
                return ((start[0], col) for col in self.cols[start[1] :: 1])
            case _:
                assert False


@dataclass(slots=True)
class Grid:
    start: Pair
    tiles: list[list[bool]]
    ranges: Ranges

    @classmethod
    def from_lines(cls, lines: Iterable[str]) -> Self:
        start = None
        coord = (0, 0)
        rows = []
        for r, line in enumerate(lines):
            row: list[bool] = []
            for c, char in enumerate(line):
                coord = (r, c)

                if char == "^":
                    start = coord
                row.append(char == "#")
            rows.append(row)

        assert start is not None
        return cls(start, rows, Ranges(range(len(rows)), range(len(rows[0]))))

    def __getitem__(self, key: Pair) -> bool:
        return self.tiles[key[0]][key[1]]


@contextmanager
def read_lines(path: Path) -> Iterator[Iterator[str]]:
    with path.open() as f:
        yield (line.rstrip() for line in f)


def walk(grid: Grid, obstruction: Pair | None = None) -> Iterator[Pair]:
    direction = up
    start = grid.start

    yield start

    while True:
        walk = pairwise(grid.ranges.walk(start, direction))

        for prev, curr in walk:
            if grid[curr] or curr == obstruction:
                direction = turn(direction)
                start = prev
                break

            yield curr
        else:
            return


def loops[T: Hashable](it: Iterator[T]) -> bool:
    visited = set()
    for e in it:
        if e in visited:
            return True
        visited.add(e)
    return False


if __name__ == "__main__":
    with read_lines(Path("inputs") / "d6.txt") as lines:
        grid = Grid.from_lines(lines)

    path = set(walk(grid))
    print("part1", len(path))

    candidates = (node for node in path if node != grid.start)
    print(
        "part2",
        sum(1 for node in candidates if loops(pairwise(walk(grid, node)))),
    )
```

A few details I want to highlight here:
- `Grid` implements `__getitem__` so we can access locations on a map more directly e.g. `grid[1, 2]` will access location (1, 2)
- `Ranges` represents the full range of locations in the map and allows us to easily trace paths in a single direction
- `walk` brings everything together and handles the logic when we hit an obstruction.

Part 2 is what we're more interested in here. We iterate for all locations on the path (around 5000 in my case) and check if adding an obstruction there forms a loop. 

## Parallelisation

I timed just part 2 and it takes around 4 seconds on my 11 core M3 Mac. However it should be simple to parallelise, as each of these walks are independent and the grid is only being read not modified. 

Using `ThreadPoolExecutor`, we can change the last part of the code to run in parallel:

```python
with ThreadPoolExecutor() as executor:
    results = executor.map(
        lambda node: loops(pairwise(walk(grid, node))),
        candidates,
    )
    print("part2", sum(1 for r in results if r))
```

I was then shocked to find that the execution was almost twice as slow at almost 10 seconds.

## Hypothesis
There are several potential reasons to explain this kind of behaviour.

### Specialization
Python 3.11 released with performance enhancements via [Specialization](https://peps.python.org/pep-0659/).

Free-threaded Python has specialisation disabled in threaded code, see https://peps.python.org/pep-0703/#improved-specialization for more details

The performance overhead should however be fairly small, according to the [PEP](https://peps.python.org/pep-0703/#performance) the slowdown should be within 10%.

### Lock contention 
The GIL's job was to protect objects from concurrent access. This is still important in free-threaded python, so there must be per-object locks in place of the GIL.

These locks can still cause threads to block waiting on them. 

### Thread start up time 
This is more of an issue associated with processes, sub-interpreters but not threads. See [Anthony Shaw's post](https://tonybaloney.github.io/posts/sub-interpreter-web-workers.html#how-do-sub-interpreters-compare-in-performance-in-real-terms). So the startup time cannot explain the slowness. 


## Profiling
To test my theories, my immediate thought was just run `cProfile`. 

```shell
python -m cProfile d6.py
```

... and nothing! The process hangs, and no output is shown. It turns out there's a [bug](https://github.com/python/cpython/issues/125165) in cProfile for free-threaded builds of Python 🤦.

Well this made the problem exponentially harder. I've tried a few other profiler options without success. I briefly considered looking into [sys.monitoring](https://docs.python.org/3/library/sys.monitoring.html) but it would probably take me a lot more research to write a profiler.

## Experiment, Research, Repeat
Without good profiling the only thing to do is to understand the problem better and try a bunch of things.

### Time per thread count
First thing I checked the performance with respect to the thread count, by default, python sets the `max_worker` count to n + 4 where n is the number of cores in the system. 

```python
for workers in range(1, 16):
    candidates = (node for node in path if node != grid.start)

    with timer(f"{workers = }: "):
        with ThreadPoolExecutor(max_workers=workers) as executor:
            results = executor.map(
                lambda node: loops(pairwise(walk(grid, node))),
                candidates,
            )
        print("part2", sum(1 for r in results if r), end="; ")
```
```shell
(advent2024-python) ➜  advent2024-python git:(main) ✗ python d6.py
part1 4883
part2 1655; workers = 1:  4.060725 s elapsed
part2 1655; workers = 2:  4.789434 s elapsed
part2 1655; workers = 3:  5.203642 s elapsed
part2 1655; workers = 4:  5.516271 s elapsed
part2 1655; workers = 5:  6.114952 s elapsed
part2 1655; workers = 6:  7.461776 s elapsed
part2 1655; workers = 7:  7.562252 s elapsed
part2 1655; workers = 8:  8.026832 s elapsed
part2 1655; workers = 9:  8.335291 s elapsed
part2 1655; workers = 10:  8.731357 s elapsed
part2 1655; workers = 11:  9.152564 s elapsed
part2 1655; workers = 12:  9.476967 s elapsed
part2 1655; workers = 13:  9.589462 s elapsed
part2 1655; workers = 14:  9.955716 s elapsed
part2 1655; workers = 15:  9.728167 s elapsed
```
So more thread means less performance here, this strongly suggests some sort of lock contention.

### How locking works
Built-in types use [internal locks](https://docs.python.org/3.13/howto/free-threading-python.html#thread-safety) to guarantee thread safety.

I read the [PEP](https://peps.python.org/pep-0703/#container-thread-safety) and it appears that the locking is rather complicated. For the most part read access is "[optimistic](https://peps.python.org/pep-0703/#optimistically-avoiding-locking)". Since we only read the list it's unlikely to be the culprit. However, to eliminate any doubts I've switched to the immutable tuple, which does not require locking:

```python
@dataclass(slots=True)
class Grid:
    start: Pair
    tiles: tuple[tuple[bool, ...], ...]
    ranges: Ranges
```

As expected this did not solve the issue and the performance degradation did not go away.

### Reference counting
This is going very deep into Python internals, not something I'm particularly comfortable with. I did get a bit of a clue [here](https://github.com/python/cpython/issues/120040#issuecomment-2152986725) where someone else also ran into performance issues.

[Reference counting](https://github.com/python/cpython/blob/main/InternalDocs/garbage_collector.md) is the method used by Python for garbage collection. The idea is that each object stores a counter for the number of references which is then used for garbage collection when the references hit 0. 

The bigger implication here is that referencing an object will require the counter to be incremented, which requires locking when the object is referenced from multiple threads.

A form of reference counting called "[biased reference counting](https://peps.python.org/pep-0703/#biased-reference-counting)" is used in free-threaded Python. This allows faster reference count access to objects created within the thread than objects created from different threads.

Overall, this means that even read-only access can be slowed down due to reference counting.

### Immortal objects
See [PEP-683](https://peps.python.org/pep-0683/), this is a recent change too that skips reference counting by marking certain objects as immortal. Immortalization is [implemented](https://peps.python.org/pep-0703/#immortalization) in free-threaded python as well. It's only really applied to `True`, `False`, `None` as well as classes and top level functions. 

Here my grid is already using booleans so it's already using immortal objects. I did use `Enum` in an earlier version and there was definitely an improvement of the performance here. But it wasn't very significant. Nevertheless it might solve performance issues in other situations.

## The Breakthrough
After many iterations, I finally looked at the following line:

```python
if grid[current] or current == obstruction:
```

This uses the magic method `__getitem__` 
```python
    def __getitem__(self, key: Pair) -> bool:
        return self.tiles[key[0]][key[1]]
```

which references the `Grid` object and gets the in turn then references the `tiles` attribute. 

The code also lives in the `walk` function inside the double loop so it's called frequently!


```python
def walk(grid: Grid, obstruction: Pair | None = None) -> Iterator[Pair]:
    ...
    while True:
        ...
        for prev, curr in walk:
            if grid[curr] or curr == obstruction:
                ...
            ...
        ...
```

> I'd like to claim that I intuitively looked here, but really I looked in a lot of different places before finding this.

In any case we can bypass the `__getitem__` call by
```python
for prev, (r, c) in walk:
    if grid.tiles[r][c] or (r, c) == obstruction:
```

Running the code again: 
```bash
(advent2024-python) ➜  advent2024-python git:(main) ✗ python d6_threads.py
part1 4883
part2 1655; workers = 1:  3.801581 s elapsed
part2 1655; workers = 2:  3.10458 s elapsed
part2 1655; workers = 3:  2.925348 s elapsed
part2 1655; workers = 4:  2.944652 s elapsed
part2 1655; workers = 5:  3.200314 s elapsed
part2 1655; workers = 6:  4.337672 s elapsed
part2 1655; workers = 7:  4.507129 s elapsed
part2 1655; workers = 8:  4.739261 s elapsed
part2 1655; workers = 9:  4.932432 s elapsed
part2 1655; workers = 10:  5.219045 s elapsed
part2 1655; workers = 11:  5.798776 s elapsed
part2 1655; workers = 12:  6.20557 s elapsed
part2 1655; workers = 13:  6.459673 s elapsed
part2 1655; workers = 14:  6.571967 s elapsed
part2 1655; workers = 15:  6.590083 s elapsed
```

This is actually the first time we've seen positive performance gain. Curiously, the higher thread counts still result in negative performance. 

Taking this further we still reference `grid` inside the loop, so we can assign tiles to the outside instead:

```python
    tiles = grid.tiles 

    while True:
        walk = pairwise(ranges.walk(start, direction))

        for prev, (r, c) in walk:
            if tiles[r][c] or (r, c) == obstruction:
                ...
```

Once again:
```shell
(advent2024-python) ➜  advent2024-python git:(main) ✗ python d6_threads.py
part1 4883
part2 1655; workers = 1:  3.572743 s elapsed
part2 1655; workers = 2:  2.257732 s elapsed
part2 1655; workers = 3:  1.665534 s elapsed
part2 1655; workers = 4:  1.38678 s elapsed
part2 1655; workers = 5:  1.383013 s elapsed
part2 1655; workers = 6:  2.540588 s elapsed
part2 1655; workers = 7:  3.520335 s elapsed
part2 1655; workers = 8:  4.365446 s elapsed
part2 1655; workers = 9:  5.072404 s elapsed
part2 1655; workers = 10:  5.722957 s elapsed
part2 1655; workers = 11:  6.087521 s elapsed
part2 1655; workers = 12:  6.179896 s elapsed
part2 1655; workers = 13:  6.174642 s elapsed
part2 1655; workers = 14:  6.194982 s elapsed
part2 1655; workers = 15:  6.100845 s elapsed
```

A decent performance when using 3, 4 or 5 workers.

### Worker count
It seems to me like there's a sweet spot for the number of cores vs the reference contention. 4 seems to be quite consistently good on my machine too cores then the reference count contention starts dominating the timing.

It's also possible that it's an indication of performance cores vs efficiency cores on the M3 processor, but I believe this is less likely as we've already demonstrated how dramatic the differences are when it comes to reference count contention.

## Key Takeaways
When looking at free threading in Python here are the following key things to look for.

### Threads are not free
Threads are not free in terms of performance, you definitely have to pay for it. And concurrent access means you can't really be free of locks.

### Avoid shared mutable writes
In this example we didn't have any shared writes, however in based on my previous testing, this can be extremely slow.

Try to structure objects so that they don't need to be mutated across threads.

### Minimise shared reads
I don't believe shared reads that can be avoided, the ease of memory access is the biggest advantage of multi-threading. 

### Worker count may be important
A direct takeaway from the idea that "threads are not free" is that you may need to reduce the number of threads based on the situation. 

It's important to test the code with different number of worker threads.

### Profiling is important
Profiling is extremely important, I hope cProfile gets fixed or some other profiler can help with this situation.


## Conclusion
There's a lot to unpack here, but my biggest thought is that multi-threading in Python is actually deceptively difficult. Simple Python operations may have an amplified performance impact. 

Debugging performance problems should get easier when we have profiling working again, so this shouldn't be a big issue for engineers looking to solve parallel problems.

However I'm a bit worried about existing multi-threaded code. Before free-threaded python, threads were used for IO bound operations when the GIL was released. It's going to be a big challenge if one day the GIL is disabled and the performance may become many times slower. 

I'm starting to think that the approach [sub-interpreters](https://peps.python.org/pep-0554/) is taking, with brand new concurrency primitives and shared memory data structures has a lot of merits. I like the idea of adding a new dimension of concurrency as opposed to modifying the current. Perhaps I'll try to implement my AOC solution using sub-interpreters.

Finally, despite the effort it took me here, I can see that a lot of work has gone into Python to make the GIL optional. From immortalization to biased reference counting and many more I haven't covered. There is definitely a lot of space for performance improvement. I can't wait to see how the framework authors [take advantage](https://peps.python.org/pep-0703/#motivation) of free-threading.
