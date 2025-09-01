Title: Simplify lambda deployments with UV.
Date: 2025-09-01
Category: Blog

The python packaging landscape and developer experience has shifted dramatically in the past year or so with [uv](https://docs.astral.sh/uv/)'s launch marked a pivotal moment. But behind the scenes many PEPs have worked to get us to this point. 

One such PEP was [PEP 723 – Inline script metadata](https://peps.python.org/pep-0723/) which we discussed in [why you should write your tools in Python Again]({filename}/cli-tools-in-python.md). This PEP combined with support from uv allow us to write single file scripts whilst also handling dependencies in the same script. 

I've been thinking about other places this might be useful outside of command line apps. Having recently used a lot of lambdas, I believe lambdas are the perfect place to use inline metadata and UV.

## Building a single file lambda with UV
Lambdas most often run a small amount of simple code meaning the code is usually a single file. This fits particularly well with inline-metadata as we can keep everything compact and simple. 

So let's give it a go, using the commands in uv's [docs](https://docs.astral.sh/uv/guides/scripts/#creating-a-python-script).

#### `uv init`
Start by creating a file to write our code, we can define which python version we want here:

```bash
uv init --script main.py --python 3.13
```
This creates a file: 

```py
# /// script
# requires-python = ">=3.13"
# dependencies = []
# ///


def main() -> None:
    print("Hello from main.py!")


if __name__ == "__main__":
    main()
```

We can now add our code to it:

```python
# /// script
# requires-python = ">=3.13"
# dependencies = []
# ///

from pydantic import BaseModel


class Body(BaseModel):
    message: str


def lambda_handler(event, context):
    return {"statusCode": 200, "body": Body(message="Hello, World!").model_dump_json()}
```

#### `uv add`
Then we add any dependencies we need: 

```bash
uv add --script main.py pydantic
```

This adds the dependency to the metadata of the file:

```py
# /// script
# requires-python = ">=3.13"
# dependencies = [
#     "pydantic",
# ]
# ///
...
```

#### Optional: `uv lock`
`uv lock` is a relatively new feature for scripts, for those who want reproducible builds using lock files.

```bash
uv lock --script main.py
```

Produces a lock file at `main.py.lock`. When the lock file is present uv will then respect the lockfile's dependency. 

#### Develop code and `uv run` locally
After you develop your code, you can run the script without any virtual environment setup using: 

```bash
uv run --script main.py
```

#### Packaging the app
So far we've used relatively well known workflows within uv. Now comes the tricky part, we need to download the dependencies and package it alongside the code. 

The simplest way I found is to first export the deps as a requirements file.
```bash 
uv export --script main.py --output-file requirements.txt
```

And then install using `uv pip install`

```bash
uv pip install \
    -r requirements.txt \
    --target package/ \
    --python-platform x86_64-manylinux2014 \
    --python-version 3.13 \
    --link-mode copy \
    --only-binary=:all: \
    --upgrade
```
Now this is a complex command! Let's break it down:

- `--target` to install the package to a directory instead of a virtualenv
- `--python-platform` is needed as we need to install packages for the lambdas specific. The options are essentially just arm or x86.
-- `--link-code` copy to produce a hard copy of the package if it comes from the uv cache
-- `--only-binary` to prevent uv from building a wheel from sdist, the resulting wheel may not work on the target platform.

We need this complexity because pydantic contains platform specific native code. This is also a problem for pip.

#### Creating the zip file and uploading it
Creating the zip file now is simple:
```bash
cd package
zip -r ../deployment.zip .
cd ..
zip -g deployment.zip main.py
```

Finally we can use aws cli to upload the zip file:
```bash
aws lambda update-function-code \
  --function-name <YOUR_FUNCTION_NAME> \
  --zip-file fileb://deployment.zip \
```

We now have a workflow that uses a single file for both code and dependencies, which is convenient and then leverages uv for its dependency management and faster workflow.


## Making things easier
But we can do a bit better, in our current workflow we still need to know which platform we need to install the dependencies for. To avoid this, we can simply just look up the platform and python versoin from AWS using `boto3`:

```py
def get_lambda_info(client: LambdaClient, name: str) -> dict:
    response = client.get_function_configuration(FunctionName=name)
    match response:
        case {"Runtime": runtime, "Architectures": ["x86_64"]}:
            return {
                "python-version": runtime.remove_prefix("python"),
                "architecture": "x86_64-manylinux2014",
            }
        case {"Runtime": runtime, "Architectures": ["arm64"]}:
            return {
                "python-version": runtime.remove_prefix("python"),
                "architecture": "aarch64-manylinux2014",
            }

        case other:
            raise Exception(f"Unexpected response: {other}")
```

## Putting things together
Since `get_lambda_info` is a python function, and if I'm honest I am terrible at bash scripting. I've added everything in a Python script and published it as [simple-lambda](https://pypi.org/project/simple-lambda/). 

I won't include the full source here, you can check it out on [github](https://github.com/Jamie-Chang/simple-lambda). 


In my script I ended up using the following builtin libraries:

- [pathlib](https://docs.python.org/3/library/pathlib.html) to handle file and directory paths.
- [tempfile](https://docs.python.org/3/library/tempfile.html) to create a temporary directory that gets deleted afterwards
- [zipfile](https://docs.python.org/3/library/zipfile.html) to bundle everything in a `.zip` file at the end

Using builtin python modules doesn't only save you from installing extra dependencies but also ensures that you code is platform-agnostic. This is a very desirable feature of a cli. 

## How to use the script
It's simple, just write your single file lambda function as before. and run:

```bash
uvx simple-lambda deploy <your_function_name> main.py
```

We're using `uvx` here which will dynamically install `simple-lambda` and run the script of the same name (by default). It also caches the package to avoid reinstalling it in the future.

If you want to explicitly keep a version of `simple-lambda` on your machine then you can install it as a tool: 

```bash
uv tool install simple-lambda
simple-lambda deploy <your_function_name> main.py
```

You do have the option to install with `pip` of course but `uv` is really just more convenient.

## Finally
My hope is that either you find the script itself useful, or you've learned a bit more about tooling in Python. 

I wrote this script to solve a real problem for myself, but I'm constantly amazed at the progress being made to improve Python packaging and tooling. 

p.s. If you want to see another great use case of inline metadata, check out [pydantic.run](https://pydantic.run/store/d49fe8ddf8c9813f).
