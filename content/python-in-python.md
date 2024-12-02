Title: Python-in-Python Sandboxing LLM Generated Code
Date: 2024-12-02
Category: Blog

I've been experimenting with [Langchain](https://www.langchain.com/) for GPT based queries. One problem we often encounter with GPT is hallucinations. This makes certain classes of problems unsuited to GPT, one example is maths and statistics. Whilst there are improvements for recent models often the maths cannot be trusted.

When I try to ask a data heavy question on https://chatgpt.com/, it generates and runs Python code and then returns the answer. I wanted to find a way to do this when using the API ideally as a langchain [tool](https://python.langchain.com/docs/concepts/tools/).


### Can we just run the code?
One solution is to run the code in a subprocess. A simple tool implementation might be:

```python
@tool
def run_python(python_code: str) -> str:
    """Run Python code and capture the results printed to stdout.

    numpy np is not available, no other 3rd party libraries are available.
    """
    return subprocess.run(["python", "-c", python_code], capture_output=True, text=True).stdout.strip()
```

So this definitely works! But it makes me feel very uncomfortable. Allowing arbitrary code to execute on a system level can be a huge security issue. If the prompt comes from a user, it's possible they can make LLM generate code that can take control or expose files on the system. 

Even if there is strict control of the prompt, we would need to manage the lifecycle of the process to make sure the computation doesn't take too long or too much resources. 


### Builtin Langchain Solutions
Looking at langchain for some help, I can see that if offers a suite of [code interpreter tools](https://python.langchain.com/docs/integrations/tools/#code-interpreter).

The interpreters are hosted as separate services, some are available only in the cloud and some can be self hosted. For cloud based interpreter there's still the question of whether you can trust the service. Especially since we're likely going to send data over. There's also the matter of cost.

The self-hosting options are certainly viable, but it's still going to require spinning up a separate Docker container.


### Python Sandbox
I wanted to find Python sandboxes that were simple to setup.

#### PyPy sandbox
The first sandbox I came across is from [PyPy](https://doc.pypy.org/en/latest/sandbox.html). With this we can apply the same `subprocess.run` command but just invoke the sandboxed version of PyPy as opposed to the regular Python interpreter. 

I've ran into problems running it:
```shell
➜  sandbox git:(main) ./pypy_interact.py     
  File "/Users/jamie.chang/personal-projects/pypy/pypy/sandbox/./pypy_interact.py", line 58
    'pypy-c': RealFile(self.executable, mode=0111),
                                             ^
SyntaxError: leading zeros in decimal integer literals are not permitted; use an 0o prefix for octal integers
```

A closer look at the docs however reveal that this might not be as actively maintained as I had hoped. There might be newer efforts of this, but I'm not sure how to access them.

#### WASM
In the past I've heard about WASM being used as a runtime which can offer isolation. For example, [cloudflare's edge workers](https://developers.cloudflare.com/workers/runtime-apis/webassembly/). [WASI](https://wasi.dev/) was introduced in large part to define and restrict what system calls are available to the wasm process. 

When researching this I came across an [article](https://til.simonwillison.net/webassembly/python-in-a-wasm-sandbox) by Simon Willison, with working code examples on how to achieve this using [wasmtime](https://github.com/bytecodealliance/wasmtime) and [VMWare's build of python.wasm](https://wasmlabs.dev/articles/python-wasm32-wasi/).

That's pretty much all the hard work done for us! 

So I've done some minor modification to Simon's code, using the latest release of `python.wasm` found [here](https://github.com/vmware-labs/webassembly-language-runtimes/releases/tag/python%2F3.12.0%2B20231211-040d5a6) the code is otherwise unchanged.


```python
def run_python_code(code: str, fuel: int = 400_000_000) -> str:
    engine_cfg = Config()
    engine_cfg.consume_fuel = True
    engine_cfg.cache = True

    linker = Linker(Engine(engine_cfg))
    linker.define_wasi()

    python_module = Module.from_file(linker.engine, "python-3.12.0.wasm")

    config = WasiConfig()

    config.argv = ("python", "-c", code)
    config.preopen_dir(".", "/")

    with NamedTemporaryFile() as out:
        config.stdout_file = out.name

        store = Store(linker.engine)

        # Limits how many instructions can be executed:
        store.set_fuel(fuel)
        store.set_wasi(config)
        instance = linker.instantiate(store, python_module)

        # _start is the default wasi main function
        start = instance.exports(store)["_start"]
        start(store)
        return out.read().decode()


@tool
def run_python(python_code: str) -> str:
    """Run Python code capturing the stdout.

    numpy np is not available, no other 3rd party libraries are available.
    """
    return run_python_code(python_code)
```
This feels much closer to the correct solution and not just because the code is already written for me.

I like that the solution relies on the `WASI` standard. This is currently a rather trendy technology and the support for it is increasing. By default `WASI` programs do not have access to anything on the filesystem, and extra permissions must be requested.

Wasmtime has also made things lot easier. The Python bindings mean that we can run the `WASM` binary inside the same Python process, without a need to create subprocesses, hence the title "Python-in-Python" or more accurately (but not as catchy) "Python-in-WASM-in-Python".

The portability of `WASM` means that the same method can be used to run Python sandbox in other languages and other platforms. There's even competing runtimes for `WASM` like [wasmer](https://github.com/wasmerio/wasmer-python).

Also I found out later that the [Riza Interpreter](https://python.langchain.com/docs/integrations/tools/riza/) with a supported langchain tool is also `WASM` based. Though it's still hosted in a separate container.

### Further Work
There are a few small caveats here. For one I only have surface knowledge when it comes to `WASM` and wasmer. There might be security concerns I haven't thought about.

Also you might have noticed that for the tool implementation, I've stated in the description that
> "numpy np is not available, no other 3rd party libraries are available."

This is because GPT has a tendency to use `numpy` whenever possible. I've not worked out exactly how to make libraries like numpy available to `python.wasm`, there is some mention of how to do this in the wasmlabs [article](https://wasmlabs.dev/articles/python-wasm32-wasi/) but it means a more complex setup process and I'm uncertain that this will work for `numpy` which contains native binary.

There's definitely more work that can be done to simplify the setup process even more. Maybe it can even be made pip installable. It might also be good to investigate working with other languages.
