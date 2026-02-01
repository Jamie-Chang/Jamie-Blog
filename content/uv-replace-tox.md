Title: Replacing tox with UV
Date: 2026-02-01
Category: Blog
Tags: Python, UV, Tutorial

Recently I've been updating some of my libraries to Python 3.10+ after Python 3.9 has finally reached end of life.

The upgrade to Python 3.10 is a relatively simple one, but I thought it be a good idea to run tests on different versions of Python.

### Existing Tooling
[tox](https://tox.wiki/en/4.34.1/user_guide.html) is a common tool for this use case, it allows you to define environments declaratively. 
```toml
requires = ["tox>=4"]
env_list = ["3.14", "3.13", "3.12", "3.11", "3.10"]

[env_run_base]
description = "run unit tests"
deps = [
    "pytest>=8",
    "pytest-sugar"
]
commands = [["pytest", { replace = "posargs", default = ["tests"], extend = true }]]
```

Similarly [nox](https://nox.thea.codes/en/stable/) let's you do the same but in a more imperative way.

### uv to rule them all
The thing is, tox and nox requires learning and using a new set of tools, whilst that's completely fine `uv` has spoiled us with the convenience of using a single tool.

There's already a lot that can be replaced by `uv`, for example:
```bash
python -m venv
uv venv
```

```bash
pip install ...
uv pip install ...
```

```bash
pyenv install ...
uv python install ...
```

I read an [article](https://til.simonwillison.net/python/uv-tests) from Simon Willison where he already worked out how to test a package with different Python versions with the [`-p` flag](https://docs.astral.sh/uv/reference/cli/#uv-run--python).

```bash
uv run -p 3.10 pytest
```

This is pretty much job done here! But we can go a bit further ...

#### Extras
You can specify any extras and dependency groups via `--extra` and `--group` respectively:

```bash
uv run -p 3.10 --extra cli pytest
```

```bash
uv run -p 3.10 --group test pytest
```

#### Overriding Package Versions
A common use of `tox` is to test over different package versions, for example, if a package needs to support both `pydantic==1.*.*` and  `pydantic==2.*.*`. We can also do this in uv with the `--with` or `-w` options:

```bash
uv run -w 'pydantic==1.*.*' pytest
uv run -w 'pydantic==2.*.*' pytest
```

This also works for a set of deps in a `requirements.txt` file:

```bash
uv run --with-requirements requirements-production.txt pytest
```

Simon's article also mentions using `--with-editable` to test with the local version of the code, which can come in handy:

```bash
uv run --with 'pydantic==1.*.*' --with-editable 'mylocal-package' pytest
```

#### Using an isolated venv
Simon also mentions using an isolated venv:
```bash
uv run --with 'pydantic==1.*.*' --isolated pytest
```
to avoid your test venv contaminating your development venv or vice versa.

I found this most useful when I needed to run tests in parallel, something like:

```bash
parallel "uv run -p {} --isolated pytest" ::: 3.{10..14}
```

This is a quick proof of concept but you can probably write a more robust bash or python script.


### Conclusion
`tox` still remains useful especially with something like [tox-uv](https://github.com/tox-dev/tox-uv). But if you're using uv anyway it may be useful to learn all the options mentioned above.

I've already had a lot of success using `uv` to quickly debug issues caused by version incompatibilities in pandas, something that just takes a bit more time to setup with tools like `tox`.