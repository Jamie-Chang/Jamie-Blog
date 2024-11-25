Title: My SQLAlchemy Cookbook  
Date: 2024-11-25
Category: Blog

I've worked with SQLAlchemy for a while now, and in my opinion it's the best ORM in Python. It's feature rich with strong support for all major databases. And it maintains the SQL feel without losing things like typing.

However there are some challenges here. Despite having very nice documentation and good abstractions for beginners, there can sometimes be an abundance of choice, which can make it more difficult to start on a problem and make it harder to collaborate. 

## Library
It's tempting to write a library on top of SQLAlchemy that includes the patterns you want to use. However it's hard to find any other reason to do so. Code in SQLAlchemy is already very concise, further abstractions won't yield any tangible benefits to size and complexity of the user code.

There are also many downsides to writing a library namely maintenance and development velocity.

## Cookbook
My preferred way is to have a single place to keep all the patterns I use.

I was inspired by the official [Python Logging Cookbook](https://docs.python.org/3/howto/logging-cookbook.html). It manages to provide code examples and explanations for logging without needing to bloat the Python standard library more than necessary. 

The cookbook is written using [JupyterLite](https://jupyter.org/try-jupyter/lab/index.html) in the web, this is a nice trick I stole from [DuckDB's blog](https://duckdb.org/2024/10/02/pyodide.html).

---

<iframe
  src="https://jamie-chang.github.io/cookbooks/notebooks/index.html?path=sqlalchemy.ipynb"
  width="100%"
  height="900em"
></iframe>
