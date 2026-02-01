Title: Webasgi - Run python web apps in a browser 
Date: 2026-01-26
Category: Blog

I recently published a project [webasgi](https://jamie-chang.github.io/webasgi/) a static site allows you run [FastAPI](https://fastapi.tiangolo.com/) and other [asgi](https://asgi.readthedocs.io/en/latest/) apps (such as [litestar](https://litestar.dev/)) right in the browser.


![webasgi]({static}/images/webasgi.png)



As you can imagine, Python web apps are not designed to run statically in the browser, and this project has been particularly challenging to me, in fact it's taken me over a year and a very productive AI coding session to get this done. 


### Vibe coding
Let's get the AI coding stuff out of the way!

I used Google's new [antigravity](https://antigravity.google/) editor with Claude Opus to do most of the coding. This is mostly a concession for me as I don't have the time or expertise in Javascript to do a lot of the work. I do feel that in some sense I robbed myself the opportunity to learn through the process of coding, however the design of the app is the most interesting and what we'll focus one in this post.

The last thing I'll say is that, this is not the first time I've attempted to use AI tools to help me code up this project, but it's the first time successful. This is probably owing to the newer models and the fact that [Pyodide](https://pyodide.org/en/stable/#) is more popular now and likely more present in the model's knowledge.

In any case, the AI discussion will have to carry on another day ... 

### [Pyodide](https://pyodide.org/en/stable/#)

I've already used [pyodide](https://pyodide.org/en/stable/#) before. I've used it in several past posts:

- [My Chinese birthdays, time keeping is hard!]({filename}/lunar-calendar.md): pyscript shell using pyodide.
- [My SQLAlchemy Cookbook]({filename}/my-sqlalchemy-cookbook.md): jupyter-lite notebook powered by pyodide
- [Trying out marimo notebooks]({filename}/fun-with-marimo.md): marimo notebook also powered by pyodide.


Pyodide is an amazing project that brings a Python to the browser using [WASM](https://webassembly.org/). The entire Python runtime is contained within the browser with no additional server required, therefore it also has the benefit of being able to run user generated Python code safely.

As such, it's already being used to create sandboxes. A prominent example is, [pydantic.run](https://pydantic.run/) which is also the inspiration of my project. 


### [ASGI](https://asgi.readthedocs.io/en/latest/) (Asynchronous Server Gateway Interface)
ASGI provides an abstraction for all Python web applications. Effectively splitting the web server into two components: 

- The application itself which in its simplest form is an async function that takes a request and returns a response. E.g. [fastapi](https://fastapi.tiangolo.com/).
- The web server that handles the connections and scaling. This comes in the form of libraries with horse sounding names such as [uvicorn](https://uvicorn.dev/), [hypercorn](https://hypercorn.readthedocs.io/en/latest/) and [granian](https://github.com/emmett-framework/granian).

This is quite a clean abstraction, allowing both sides to evolve independently, and giving the user a good amount of flexibility. One of my favourite consequences of ASGI is that we can quite easily compose ASGI applications, as seen [here](https://jamie-chang.github.io/webasgi/?auto=true#code/eyJtYWluLnB5IjoiIyAvLy8gc2NyaXB0XG4jIHJlcXVpcmVzLXB5dGhvbiA9IFwiPj0zLjExXCJcbiMgZGVwZW5kZW5jaWVzID0gW1xuIyAgIFwiZmFzdGFwaVwiLFxuIyAgIFwibGl0ZXN0YXJcIixcbiMgXVxuIyAvLy9cbmZyb20gcGF0aGxpYiBpbXBvcnQgUGF0aFxuXG5mcm9tIGZhc3RhcGkgaW1wb3J0IEZhc3RBUElcbmZyb20gZmFzdGFwaS5yZXNwb25zZXMgaW1wb3J0IEhUTUxSZXNwb25zZVxuaW1wb3J0IGxpdGVzdGFyX2FwcFxuXG5cbmFwcCA9IEZhc3RBUEkoKVxuXG5AYXBwLmdldChcIi9cIiwgcmVzcG9uc2VfY2xhc3M9SFRNTFJlc3BvbnNlKVxuYXN5bmMgZGVmIGhvbWUoKTpcbiAgICByZXR1cm4gKFBhdGguY3dkKCkgLyBcImluZGV4Lmh0bWxcIikucmVhZF90ZXh0KClcblxuXG5AYXBwLmdldChcIi9hcGlcIilcbmFzeW5jIGRlZiBhcGkoKTpcbiAgICByZXR1cm4ge1wibWVzc2FnZVwiOiBcIkhlbGxvIGZyb20gRmFzdEFQSSFcIiwgXCJmcmFtZXdvcmtcIjogXCJGYXN0QVBJXCJ9XG5cblxuYXBwLm1vdW50KFwiL3N1YmFwcFwiLCBsaXRlc3Rhcl9hcHAuYXBwKVxuIiwiaW5kZXguaHRtbCI6IjwhRE9DVFlQRSBodG1sPlxuPGh0bWw-XG48aGVhZD5cbiAgICA8dGl0bGU-TmVzdGluZyBBU0dJIEFwcHM8L3RpdGxlPlxuICAgIDxzdHlsZT5cbiAgICAgICAgYm9keSB7IGZvbnQtZmFtaWx5OiBzeXN0ZW0tdWk7IG1heC13aWR0aDogNjAwcHg7IG1hcmdpbjogNTBweCBhdXRvOyBwYWRkaW5nOiAyMHB4OyB9XG4gICAgICAgIGgxIHsgY29sb3I6ICMwMDk2ODg7IH1cbiAgICAgICAgY29kZSB7IGJhY2tncm91bmQ6ICNlMGYyZjE7IHBhZGRpbmc6IDJweCA2cHg7IGJvcmRlci1yYWRpdXM6IDRweDsgfVxuICAgICAgICBwcmUgeyBiYWNrZ3JvdW5kOiAjZjVmNWY1OyBwYWRkaW5nOiAxMnB4OyBib3JkZXItcmFkaXVzOiA4cHg7IG92ZXJmbG93LXg6IGF1dG87IH1cbiAgICA8L3N0eWxlPlxuPC9oZWFkPlxuPGJvZHk-XG4gICAgPGgxPk5lc3RpbmcgQVNHSSBBcHBzPC9oMT5cbiAgICA8cD5EbyB5b3Ugb2Z0ZW4gZmluZCB5b3Vyc2VsZiBzdHJ1Z2dsaW5nIHRvIGRlY2lkZSBvbiBmcmFtZXdvcmtzIHRvIHVzZT88L3A-XG4gICAgPHA-T3IgeW91IG1pZ2h0IGFscmVhZHkgaGF2ZSBhbiBhcHAgd3JpdHRlbiBpbiBGYXN0QVBJIGJ1dCB3b3VsZCBsb3ZlIHRvIHRyeSBvdXQgbGl0ZXN0YXIgZm9yIGEgbmV3IGZlYXR1cmUuPC9wPlxuICAgIDxwPkx1Y2tpbHkgeW91IGRvbid0IGhhdmUgdG8gY2hvb3NlIGFueSBtb3JlLCBib3RoIEZhc3RBUEkgYW5kIGxpdGVzdGFyIGZlYXR1cmUgd2F5cyB0byBtb3VudCBzdWJhcHBsaWNhdGlvbnM6PC9wPlxuICAgIDx1bD5cbiAgICAgICAgPGxpPjxhIGhyZWY9XCJodHRwczovL2Zhc3RhcGkudGlhbmdvbG8uY29tL2FkdmFuY2VkL3N1Yi1hcHBsaWNhdGlvbnMvI21vdW50aW5nLWEtZmFzdGFwaS1hcHBsaWNhdGlvblwiPkZhc3RBUEkgbW91bnRpbmcgYW4gYXBwbGljYXRpb248L2E-PC9saT5cbiAgICAgICAgPGxpPjxhIGhyZWY9XCJodHRwczovL2RvY3MubGl0ZXN0YXIuZGV2LzIvdXNhZ2Uvcm91dGluZy9vdmVydmlldy5odG1sI21vdW50aW5nLWFzZ2ktYXBwc1wiPkxpdGVzdGFyIGVxdWl2YWxlbnQ8L2E-PC9saT5cbiAgICA8L3VsPlxuXG4gICAgPHByZT5cbiAgICBmcm9tIGZhc3RhcGkgaW1wb3J0IEZhc3RBUElcblxuICAgIGFwcCA9IEZhc3RBUEkoKSAjIG9yIG90aGVyIEFTR0kgYXBwcyBsaWtlIGxpdGVzdGFyXG4gICAgYXBwLm1vdW50KFwiL3N1YmFwcFwiLCBzdWJfYXBwKVxuICAgIDwvcHJlPlxuXG4gICAgPHA-TGl0ZXN0YXIgaXMgYSBsaXR0bGUgbW9yZSB2ZXJib3NlPC9wPlxuICAgIDxwcmU-XG4gICAgYXBwID0gTGl0ZXN0YXIoW1xuICAgICAgICBhc2dpKHBhdGg9XCIvc3ViYXBwXCIsIGlzX21vdW50PVRydWUsIGNvcHlfc2NvcGU9VHJ1ZSkoc3ViX2FwcClcbiAgICBdKVxuICAgIDwvcHJlPlxuXG4gICAgPHA-RG9uJ3QgYmVsaWV2ZSBtZT8gdHJ5IDxjb2RlPjxhIGhyZWY9XCIvc3ViYXBwL1wiPi9zdWJhcHAvPC9hPjwvY29kZT4gYW5kIDxjb2RlPjxhIGhyZWY9XCIvc3ViYXBwL3N1YmFwcC9cIj4vc3ViYXBwL3N1YmFwcC88L2E-PC9jb2RlPjwvcD5cblxuICAgIDxwPlRoaXMgaXMgYSBsZXZlbCBvZiBmbGV4aWJpbGl0eSBtYWRlIHBvc3NpYmxlIGJ5IHRoZSBBU0dJIHN0YW5kYXJkITwvcD5cblxuPC9ib2R5PlxuPC9odG1sPiIsImxpdGVzdGFyX2FwcC5weSI6ImZyb20gbGl0ZXN0YXIgaW1wb3J0IExpdGVzdGFyLCBhc2dpLCBnZXRcblxuaW1wb3J0IGlubmVyX2Zhc3RhcGlcblxuXG5AZ2V0KFwiL1wiKVxuYXN5bmMgZGVmIGhlbGxvX3dvcmxkKCkgLT4gc3RyOlxuICAgIHJldHVybiBcIkhlbGxvLCBmcm9tIGxpdGVzdGFyIVwiXG5cblxuXG5hcHAgPSBMaXRlc3RhcihbXG4gICAgaGVsbG9fd29ybGQsXG4gICAgYXNnaShwYXRoPVwiL3N1YmFwcFwiLCBpc19tb3VudD1UcnVlLCBjb3B5X3Njb3BlPVRydWUpKFxuICAgICAgICBpbm5lcl9mYXN0YXBpLmFwcFxuICAgIClcbl0pXG4iLCJpbm5lcl9mYXN0YXBpLnB5IjoiZnJvbSBmYXN0YXBpIGltcG9ydCBGYXN0QVBJXG5cbmFwcCA9IEZhc3RBUEkoKVxuXG5AYXBwLmdldChcIi9cIilcbmFzeW5jIGRlZiBoZWxsb193b3JsZCgpOlxuICAgIHJldHVybiBcIkhlbGxvLCBmcm9tIGlubmVyIGZhc3RhcGlcIlxuIn0) with a FastAPI app that mounts a Litestar app under a subpath. 

The server's implementation can also be very flexible, [uvicorn](https://uvicorn.dev/) and [hypercorn](https://hypercorn.readthedocs.io/en/latest/) are both Python implementation where [granian](https://github.com/emmett-framework/granian) is written in rust. 

Naturally a Pyodide implementation is also possible, but instead of connecting to a port in the operating system, we can mock the 'networking layer' in browser javascript. 

### Browser Emulation
Having worked out the basics of the Pyodide "server" the next step is to display the contents returned by the server. This is necessary as it's part of a standard FastAPI workflow: 

- write some FastAPI code.
- run the code with a server like Uvicorn.
- check the server by calling the endpoint in your browser. (often using swagger docs)

Without a clean way to visualise the output, it's always going to feel a little awkward. 

Emulating a browser means taking html which may contain linked javascript and css loading everything and rendering it. This all needs to take place without leaking into the parent browser session, that is, we don't want the emulated browser to crash out current tab.

Luckily this is exactly what [iframe](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe) does and we can pass the html as [`srcDoc`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLIFrameElement/srcdoc). 

#### The hard part ...
We've figured out how to render the responses however we're only just getting started. 

The rendered html page can contain elements that make further requests to the server. These can be relative anchor links:

```html
<a href="/other-endpoint">go to next</a>
```

Or some non-get requests via javascript or form submission. Without changing anything, these requests will not be able to go to the ASGI server, breaking the "immersion".

If we want everything to work as a browser would, we'd need to load some javascript first that patches all of these requests. This is done by injecting some scripts in the html that register javascript event handlers to intercept these events. The iframe has the ability to them securely send a message to the parent via [`postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)

### Everything put together
![Full architecture]({static}/images/mermaid2.png)

This is roughly what the full architecture looks like, the rest is mind numbing javascript implementation you can check out [here](https://github.com/Jamie-Chang/webasgi).

### Why?
So why did we do all this?

Being able to run more things in the browser dramatically lowers the barrier to entry. A functional app can be created iterated and shared in the browser. 

My immediate next steps are to improve the user experience using AI or otherwise but also to try and tidy up the AI generated parts of the code.

But my long term goal for this is to:

1. teach and discuss more advanced patterns in these web app frameworks
2. encourage adoptions of new frameworks that solve some of my headaches with frameworks like fastapi.

