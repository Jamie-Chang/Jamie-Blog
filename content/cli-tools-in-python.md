Title: Why you should write scripts in Python Again
Date: 2025-04-08
Category: Blog

## Why not in Python?
You're probably thinking that there are plenty of popular tools written in Python. But most of them are tools built for Python development like `mypy` and `flake8`, where there's already a Python dev environment setup. In contrast, general purpose cli tools tend to be built in new languages like `rust` and `go`, for example, docker cli is currently written in go.

This wasn't always the case, let's set our mind to the era of Python 2 specifically 2.7. 2.7 enjoyed an almost 10 year lifespan whilst the Python community was working on Python 3. The stability of 2.7 meant that it was easy to distribute tools as Python scripts. 

Then Python 3 arrived with lot's of nice stuff: `f-strings`, `dataclasses`, `pattern-matching` etc.. But no more stability, a script working on 3.5 might have problems on 3.7 and so on. 3rd party dependencies adding more dimension to the problem. 

Compiled languages don't tend to have this problem as the binary executable is distributed, the executable no longer depends on the language itself, but on the platform it's running on. So tools must be built for every platform it's expected to run on. This turns out to be a relatively manageable number, though recent advancements in CPU architecture (apple silicon) has made it more complex.

## Writing a terminal app
Fast forward to the present. I've been keen on writing scripts to automate some of the more mundane tasks. I wanted to write a terminal app to display my JIRA tickets, so I don't have to deal with the clunky web UI. 

I've' heard a lot of good things about [textual](https://github.com/Textualize/textual), whilst lacking in patience to learn another language, I decided to give it a go.

I proudly present my little TUI (Terminal UI):

![better-functools]({static}/images/jira-list.png)

<style type="text/css">
  .gist-file
  .gist-data {max-height: 500px;}
</style>

<details>

  <summary><i>Source code here</i></summary>
  <script src="https://gist.github.com/Jamie-Chang/644db95fc536506d301920f3c9f46da8.js"></script>
</details>

The app is very simple, just displaying the tickets and title. That's kind of the point really, too much detail and we're no better than the web UI.

I won't go into too much depth on the code itself, textual worked well as was fun to use. Instead let's discuss the massive improvements to scripting in Python.

## So how has Python fixed its problems?
In short, [uv](https://github.com/astral-sh/uv) came to the rescue. 

uv has been all the rage lately, for [good reasons](https://www.youtube.com/watch?v=8UuW8o4bHbw). Thanks to uv package management got a lot better.

In my specific case I used uv for the following.

- manage dependencies such as Python version and packages
- execute the script in an isolated Python environment without any hassle.

Let me elaborate, at the top of my script we have:

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.12"
# dependencies = [
#     "requests",
#     "textual",
#     "platformdirs",
#     "tomli-w",
#     "keyring",
# ]
# [tool.uv]
# exclude-newer = "2025-04-09T00:00:00Z"
# ///
```

[PEP-723](https://peps.python.org/pep-0723/) inline metadata creates a standard to setting the metadata like dependencies at the top of the file.

Breaking down each line we have `requires-python = ">=3.12"` for the python version and then the list of  `dependencies`.

Finally a uv specific [`exclude-newer`](https://docs.astral.sh/uv/reference/settings/#exclude-newer) that acts as a pseudo lockfile, limiting dependencies by date instead of by version.

This is a whole Python project in one, and uv will happily run it with a single command:

```bash
$ uv run --script jira-tui.py
```

Note the shebang `#!/usr/bin/env -S uv run --script`, a neat trick from [Rob Allen](https://akrabat.com/using-uv-as-your-shebang-line/). On linux the shebang line will save us from typing `uv run --script` every time.

Essentially uv has become the one stable runtime, as long as the user has uv scripts may be distributed easily.

## Python's Broad Compatibility
Just distributing the script is not enough, we also need to make sure that the end user's platform can execute the script.

Owing to Python's popularity, most packages support all major OS's and CPU architectures out of the box.

In fact, it's so comfortable you hardly have to think about it:

- textual: the main framework was designed from the ground up to be as compatible as possible
- keyring: used to store credentials uses different secure storage based on the OS.
- standard lib packages like `pathlib` and `webbrowser` all work cross platform.

The one time I actively considered platforms was where to store the config file, which was simply solved using the [`platformdirs`](https://pypi.org/project/platformdirs/) package.

## How does python compare to other languages?
The `rust` and `go` tools are definitely here to stay. In fact, `uv` is almost completely written in `rust`. The main reason to use other languages are performance and concurrency. In my (biased) experience performance is not a priority for terminal applications, but your experiences may vary.

Python's strength lies in the fact that it's simple, easy to get started with and iterate on. This applies even more so with cli scripting, where we generally one quick changes, small feedback loop and to not think about releases. 

The takeaway here is that whilst `uv` doesn't remove all of the trade-offs, it has level the playing field and made scripting fun again.
