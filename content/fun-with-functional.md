Title: better-functools: Python functional fun
Date: 2025-03-09
Category: Blog

I recently put some effort into creating [better-functools](https://github.com/Jamie-Chang/better-functools). It's a package that adds some tooling for functional programming in Python. And allows us to write some unique looking code in a manner similar to functional languages:

![better-functools]({static}/images/better-functools.png)
[Try in sandbox](https://pydantic.run/store/a075fad4d27ad9d0)

### Why? 
At a glance this doesn't look very "Pythonic". More complex expressions and syntax are often controversial in Python, but I don't think it's a good enough reason to avoid it. Python is a multi-paradigm language, but it can be awkward to do anything but the expected OOP. New syntax can really help with expanding what's possible with Python. A good example of new syntax is pattern matching, which is a feature that came from functional languages. I'm keen on using this to make functional programming more available to a wider user base, as opposed to the stereotypical functional bros.

### Inspiration
If you can't tell already, this is largely inspired by ML specifically [OCaml](https://ocaml.org/). Last Christmas I decided to solve [Advent of Code](https://github.com/Jamie-Chang/advent2024/tree/main) in OCaml and I loved it.

The biggest takeaway from this experience is that you don't need to understand monads or the other mathematical concepts to program functionally. Those are probably things you come around later but it's not make functional programming fun.

What I loved most is the the system and ergonomics in OCaml. 
> Note: The ML family are not the only programming languages, other languages may have a stronger focus on ergonomics than type safety.

OCaml is strongly typed but does not require explicit annotations, instead types are inferred.
For example, the signature of add in the following snippet is automatically inferred as `+` operator only takes integer values.

```ocaml
let add a b = a + b
(* val add : int -> int -> int = < fun > *)
```
This greatly reduces the burden on programmers. 


On the topic of ergonomics, OCaml like many other functional languages has a lot of features to make composing functions and chaining function calls easier. The most notable of which are currying:

```ocaml
let add a b = a + b
let inc = add 1
```


And [pipeline](https://cs3110.github.io/textbook/chapters/hop/pipelining.html):

```ocaml
[1; 2; 3; 4] |> map inc |> sum |> string_of_int |> print_endline
```

The implication of this is that you rarely need to abstract out functions because you can easily compose and chain them inline when you need them.


### Functional patterns with `better-functools` and Python
So how can we use these ideas in Python?

#### Currying
We can also do currying in Python, but it's awkward as Python has many different types of arguments: defaulted, positional, keyword and combinations of all of them.

Instead I've added `@bind(...)` as a short hand to bind an argument and always return a partial.

```python
inc = add @ bind(1)
```

The resulting expression is a bit more verbose than currying but has the same look and feel.

#### pipeline
The ['pipeline'](https://cs3110.github.io/textbook/chapters/hop/pipelining.html) operator `|>` in OCaml chains operations together in a clear and concise way. And I'm definitely not the only person to think this way:

- [TC39](https://github.com/tc39/proposal-pipeline-operator) is a proposal to add `|>` in JS.
- There are other python libraries that implement similar ideas e.g. [pipe](https://pypi.org/project/pipe/).
- In golang a [rejected proposal](https://github.com/golang/go/issues/68534)

My approach uses the `|` operator in python and you must explicitly start and end a pipeline
```python
Pipeline(inputs) | ... | Pipeline.unwrap
```

> The specific reason why I chose to make the `Pipeline` explicit but not `@` operations is mainly due to how common `|` is vs `@` in Python. Something I'll go in more depth in a future post. 

#### Other Cool Patterns
```python
# Binding partially to the second argument not the first
func(itertools.combinations @ func.arg(Iterable[int]) @ bind(2))

# Null coalescing 
nvl(count_or_none @ nvl @ apply(add @ bind(1)))  # increment count or return null

# Composing functions inline
sum @ compose(eq @ bind(2020))  # check sum equal to 2020 

# Mixing `|` and `@`
_ = (
    Pipeline(inputs)
    | map @ bind(mul @ 2)  # multiply all inputs by 2
    | sum  # add them up
    | eq @ bind(2020)  # check if they sum to 2020
    | print  # print the result
)
```

### Further Reading
Overall it wasn't easy but I did learn a lot about Python's operators and type system. Don't worry if you'd like to know more about these topics, I'll have follow up more posts.

Here are some further reading material about functional programming:

- Official Python documentation on functional programming in Python [here](https://docs.python.org/3/howto/functional.html)
- [A tour of OCaml](https://ocaml.org/docs/tour-of-ocaml) to get started with OCaml
- [Dear Functional Bros](https://www.youtube.com/watch?v=nuML9SmdbJ4) an excellent video on functional programming and its relationship with OOP.