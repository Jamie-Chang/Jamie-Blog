Title: Python 3.14: State of free threading
Date: 2025-05-26
Category: Blog
Tags: Python, πthon, Python3.14, Free-threading, Subinterpreters

In my posts earlier this year I talked about the parallelism performance on 3.13 free-threaded builds. In particular I looked at solving an advent of code [problem](https://adventofcode.com/2024/day/6). In [How free are threads in Python now?]({filename}/free-threading-aoc.md) I discovered significant performance penalties for using free-threading and a lack of tooling available to debug these issues. In a later [post]({filename}/subinterpreter-aoc.md) I compared free-threading with the yet unreleased subinterpreters. 

The conclusion being that subinterpreters are often overlooked but they provide far more predictable scaling characteristics. And with explicit memory management the performance is better than free-threading.

# 3.14 Updates
Python 3.14 is still in beta so everything should be taken with a grain of salt here. But there are good [news](https://docs.python.org/3.14/whatsnew/3.14.html#whatsnew314-free-threaded-cpython), in 3.14 many of the "TODO"s have been completed. This should mean better performance and fewer bugs.

So I reran my code [here](https://github.com/Jamie-Chang/advent2024-python) with 3.14:

![interpreters vs threads 3.14]({static}/images/threading-3-14.png)

Raw data:

| Worker Count | 3.14t threads | 3.13t threads | 3.14t subinterpreters | 3.13t subinterpreters | 3.14 subinterpreters | 3.13 subinterpreters |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | 2.920236 | 3.591980 | 2.848517 | 3.581693 | 3.285170 | 3.616936 |
| 2 | 1.694204 | 2.322360 | 1.673501 | 1.801588 | 1.696457 | 1.837193 |
| 3 | 1.289783 | 1.678028 | 1.042733 | 1.262571 | 1.212331 | 1.285815 |
| 4 | 1.157097 | 1.387227 | 0.828986 | 0.976082 | 0.897258 | 0.972027 |
| 5 | 1.295146 | 1.356140 | 0.704804 | 0.820581 | 0.825292 | 0.800690 |
| 6 | 2.109609 | 2.562865 | 0.719364 | 0.783600 | 0.724958 | 0.741971 |
| 7 | 2.687946 | 3.488201 | 0.642275 | 0.778133 | 0.661795 | 0.688237 |
| 8 | 3.101248 | 4.438853 | 0.645746 | 0.735963 | 0.634333 | 0.648535 |
| 9 | 3.634345 | 5.401229 | 0.649026 | 0.732540 | 0.614862 | 0.729464 |
| 10 | 4.235933 | 6.152377 | 0.712666 | 0.789292 | 0.612465 | 0.593892 |
| 11 | 4.315591 | 6.908039 | 0.682817 | 0.721079 | 0.586761 | 0.586010 |
| 12 | 4.361890 | 6.873490 | 0.653831 | 0.772412 | 0.590408 | 0.588454 |
| 13 | 4.597265 | 6.815938 | 0.644710 | 0.753303 | 0.605583 | 0.603435 |
| 14 | 4.582979 | 7.090147 | 0.751220 | 0.775110 | 0.612578 | 0.588413 |
| 15 | 4.501879 | 6.522076 | 0.725688 | 0.859769 | 0.610816 | 0.594174 |


# Better but is it enough?
Performance numbers of free-threading in 3.14 are definitely better then 3.13, but the scaling is definitely still disappointing.

I also ran different versions of subinterpreters, which were pretty much the same with small but consistent variations. Subinterpreters still generally beat free-threading.

Additionally there is still no way to profile and debug free threading performance which is a bit disappointing. 

# Let's look forward
The update may not be a game changer yet, but the trend is still positive.

There's also been talk of more radical changes to memory ownership and data structures see [Project Verona](https://microsoft.github.io/verona/pyrona.html).

I have experienced cases where free-threading hits the mark of performance whilst being easier to use compared to subinterpreters. Though the community hasn't quite worked out the direction of free-threading, there's a good chance these incremental improvements will add up in the future allowing free-threading to become a no-brainer.

For now the best thing to do is to experiment and find examples where free-threading helps and examples where it doesn't.
