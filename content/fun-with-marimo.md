Title: Trying out marimo notebooks
Date: 2025-12-03
Category: Blog

As we approach the holiday seasons, we get to "enjoy" another 12 days of [Advent of Code](https://adventofcode.com/). Every year I try to do something a little different, last year I had a lot of fun solving it in Ocaml, prior years I've tried the latest Python features. This year I've decided to use [marimo notebooks](https://marimo.io/). 

Marimo is a jupyter competitor, providing an interactive environment to develop code and visualise data. As the new kid in the block, it has a very sleek look and good support for a lot of different visualisation frameworks. There is already a very good [article](https://realpython.com/marimo-notebook/) on realpython about it, so I won't go into too much detail on its features.


Instead I'll talk about the feature that makes marimo truly standout: builtin reactivity! When a cell in marimo creates a variable, marimo tracks it. When the variable changes, any cell that references it will update automatically. For example, try changing `my_value` in the second cell below:

<iframe
  src="https://marimo.app/l/zluj3i?embed=true&show-chrome=false"
  width="100%"
  height="350"
  sandbox="allow-scripts allow-same-origin"
></iframe>

> If you're not familiar with embedded notebooks, you can have a look at my past [blog post]({filename}/my-sqlalchemy-cookbook.md) which uses jupyter-lite. Marimo has even better support for wasm notebooks.

Honestly, when you first try it everything feels like magic. But there are some quirks the more you get into it. In order for marimo to effectively track the variables, it places restrictions on where variables can be used. And cyclical dependencies are possible here.

The good thing is marimo will let you know immediately why, so you won't need to look up why something is not working. Overall I don't find it too difficult to use but it'll be good to hear what others think.

To me marimo is just much more fun than jupyter, I can essentially write little reactive web UI in pure Python. I think that's important, I'm not someone who has enjoyed writing UIs, and I think it's because I just don't find javascript fun. Of course, we're not going to be able to create particularly complex and performant UIs but I think fun matters a lot too.

In any case, here's a neat little app demonstrating the solution for today's advent of code, I've included some input UI elements like the [slider](https://docs.marimo.io/api/inputs/slider/). Changing the inputs will automatically cause the output to regenerate:


<iframe
  src="https://marimo.app/l/x8qcbz?embed=true&show-chrome=false"
  width="100%"
  height="800"
  sandbox="allow-scripts allow-same-origin"
></iframe>

So what do you think? If it looks cool to you, you can try it out at [marimo.app](https://marimo.app)!
