Title: t-strings: the good and the ugly
Date: 2025-05-07
Category: Blog
Tags: Python, πthon, Python3.14, t-strings


This one's hot off the press as the first beta for Python 3.14 (aka. π-thon) has hit. We're looking at a chunky release with a lot of new features. But  all I can think about are these new template strings (officially `t-strings`). 

[PEP-750](https://peps.python.org/pep-0750/) officially introduces the concept. The idea is the syntax of `f-strings` but allowing customised behaviours.

## Quick Recap of f-strings
F-strings were introduced in Python 3.7. The allow strings to formatted for concisely:

```python
name = "world"
f'Hello {name}'

# instead of 
'Hello {name}'.format(name=name)
```

There are also some lesser known but very useful features.

### f-string with `=`
Very useful for debugging, you can render both the name of the variable and the value with an `=` between. 

```python
name = "Jamie"
assert f'User {name = }' ==  'User name = Jamie'
```


### f-string with formatting spec
We can also quickly format objects with f-strings, this is especially useful for formatting `datetime`s:

```py
cost = 15
assert f'{cost:.2f}' == '15.00'
assert f'{datetime(2025, 1, 1):%Y/%m/%d}' == '2025/01/01'
```

## Why t-strings?
For me there are two big reasons for `t-strings` to exist.

- `f-string` syntax but with custom rendering logic.
- `f-string` but with deferred rendering.

In the PEP there are proposals to use this for `html` rendering and escaping or `sql` parameter substitution.

```py
evil = "<script>alert('evil')</script>"
template = t"<p>{evil}</p>"
assert html(template) == "<p>&lt;script&gt;alert('evil')&lt;/script&gt;</p>"
```
Where the t-string returns a `string.templatelib.Template` object and the `html` function contains the custom logic for rendering the template. In this case we delay rendering the string and then our custom logic escapes any potentially harmful variables in the string.

## How does it work?
According to the PEP, the `Template` object looks like: 
```python
class Template:
    ...  # Omitted a bunch of other fields

    def __iter__(self) -> Iterator[str | Interpolation]:
        ...
```

We can iterate the template for all the parts of the template. For a template of `t'Hello {name}!'` becomes:

```py
('Hello', Interpolation('jamie', 'name', None, ''), '!')
```

## My use cases 
I can see a lot of potential use cases many of them outlined in the PEP already.

### Logging - a decent use case 
My favourite use case so far is to solve a bit of a nit I have. The following code looks pretty innocent: 

```py
logging.debug(f"{user} - Something happened")
```

`f-strings` are eager, so `user` is stringified immediately. This is fine for simple variables but might be expensive for large nested objects. Then consider running this code in production where we turn off `debug` logging, we would still need to pay the cost of calling `str(user)` even if we don't log. This is the rationale behind [ruff's G004](https://docs.astral.sh/ruff/rules/logging-f-string/#logging-f-string-g004).


`t-strings` by nature is 'deferred', so as long as we teach logging how to handle them they will work better than `f-string` here. 

The following snippet does exactly that:
<style type="text/css">
  .gist-file
  .gist-data {max-height: 500px;}
</style>

<script src="https://gist.github.com/Jamie-Chang/65466a5e95832f81b2ffbb0208cc11c2.js"></script>


Now we can directly use `t-strings` for logging as below:

```py
initialize()

name = "Jamie"
logging.info(t'{name = }')
```



### A little ugly: shorthand kwargs
So this is where I think we can take this syntax a little too far. I had this idea a few weeks ago when [PEP-736](https://peps.python.org/pep-0736/) was rejected. PEP-736 comes from the observation that there are often redundant writing when it comes to invocation by keyword arguments: 

```py
User(name=name, birthday=birthday, ...)
```

This is actually something our `t-strings` can help with. The name of the variable and the value are captured simultaneously, so we just need to then turn the `t-string` into a dictionary:

```py
from string.templatelib import Interpolation, Template


def kw(*templates: Template) -> dict[str, object]:
    return {
        inter.expression: inter.value 
        for template in templates 
        for inter in template.interpolations
    }
```

Then we can apply it to a function call:

```py
@dataclass
class User:
    name: str
    birthday: date

User(**kw(t"{birthday} {name}"))
User(**kw(
    t"{birthday=}",
    t"{name=}",
))

```

Honestly, I'm a little proud of this, but equally repulsed. I think it demonstrates one of the problems with `t-strings`. It's the fact that they are really not strings to begin with.

They are an intermediate representation that can turn into strings, but there's no reason it has to be. 

Fringe use cases like this are usually my forte, but here it looks kind of ugly and awkward to me. This coupled with the fact that we lose all typing information when we build our 't-string` make it hard to recommend here.

## Should we be using t-strings then? 
I think `t-strings` are amazing and generally will be beneficial, I was definitely hyped about them. 

I have recently shifted my opinions on new syntax in Python. I used to hate any weird syntax, going so far as to prefer `dict.update(new_dict)` over the new `dict |= new_dict`. But my recent experiments with [DSLs]({filename}/dsl-with-operators.md) showed me that done right, there is a place for new syntax to be very beneficial.

I do also see the argument against new syntax. Before writing this post, I watched anthonywritescode's [video](https://www.youtube.com/watch?v=_QYAoNCK574&t=1640s). He actually goes very in depth on `t-strings` made a good argument about the added mental overhead of this.

Therefore I can only say that we need to find our balance here. For me, I'll pursue opportunities to use `t-string` to generate strings. But will tread more carefully when going off-piste to create complex objects.
