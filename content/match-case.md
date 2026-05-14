Title: Don't make Python code decisions based on whitespace
Date: 2026-03-14
Category: Blog

I recently saw Trey Hunner's take [blog post](https://www.pythonmorsels.com/switch-case-in-python/) on 'switch case' like statements in Python. He makes a lot of good points and it's definitely worth a read.

What I want to talk about is his last point
> Using a dictionary instead of switch-case: Sometimes you can replace long if statements with dictionaries

This is the idea of replacing something like: 
```py
def get_status_message(code):
    if code == 200:
        return "OK"
    elif code == 201:
        return "Created"
    elif code == 400:
        return "Bad Request"
    elif code == 401:
        return "Unauthorized"
    elif code == 404:
        return "Not Found"
    elif code == 500:
        return "Internal Server Error"
    else:
        return "Unknown Status"

```
with a dictionary mapping:

```py
STATUS_MESSAGES = {
    200: "OK",
    201: "Created",
    400: "Bad Request",
    401: "Unauthorized",
    404: "Not Found",
    500: "Internal Server Error"
}

def get_status_message(code):
    return STATUS_MESSAGES.get(
        code,
        "Unknown Status",
    )
```

The author argues that in this case `get_status_message` is acting more like a mapping so a dictionary is a better and more compact fit. 

People often claim that this pattern is more 'pythonic', but I think it's very subjective.  For one thing, Python is not a language that values 'compactness', if we did we would just write the `if` and `elif` branches in one line like this:

```py
    if   code == 200: return "OK"
    elif code == 201: return "Created"
    elif code == 400: return "Bad Request"
    ...
```

Whilst I would use the dictionary mapping pattern from time to time, I just don't think there's anything wrong with the `if` ... `elif` ... or an equivalent `match` ... `case` solution:

```py
    match code:
        case 200:
            return "OK"
        case 201:
            return "Created"
        case 400:
            return "Bad Request"
        ...
```
Which I feel is pretty clean too despite the amount of white spaces.

> Whitespace is not trying to hurt you, please don't let the spacing influence your decisions!

## Where you don't want dictionaries
For sure the dictionary example above is pretty darn clean. But there are places where using a dictionary is detrimental.

### Exhaustiveness
Let's say instead you are mapping an enum, or a finite set of literal values. And you want to map each value. For example:

```py
class Status(Enum):
    SUCCESS = auto()
    FAILURE = auto()
    IN_PROGRESS = auto()

STATUS_COLOR_MAP = {
    Status.SUCCESS: "green",
    Status.FAILURE: "red",
    Status.IN_PROGRESS: "yellow",
}
```

But wait, what if we add a new `Status.ABANDONED` to the enum? Well, you'll need to update the mapping or you risk getting an error.

We can actually rewrite it like so: 

```py
from typing import assert_never


def map_status_to_color(status: Status) -> str:
    match status:
        case Status.SUCCESS:
            return "green"
        case Status.FAILURE:
            return "red"
        case Status.IN_PROGRESS:
            return "yellow"
        case other:
            assert_never(other)
```

> Note: we can write pretty much the same with an if ... else statement. 

The key here is the `assert_never(other)`. This line will raise an `AssertionError` at runtime but also creates a static typing error. 

How this works is as we go through the different branches, the possible value of status reduces. This is called 'type narrowing': 

```py
case Status.SUCCESS:  # Possible: {Status.SUCCESS, Status.FAILURE, Status.IN_PROGRESS}
    return "green"
case Status.FAILURE:  # Possible: {Status.FAILURE, Status.IN_PROGRESS}
    return "red"
case Status.IN_PROGRESS:  # Possible: {Status.IN_PROGRESS}
    return "yellow"
case other:  # Possible: {} aka. the Never type
    assert_never(other)
```
The final catch-all case has no more possible type / value left, so we call it the [`Never` type](https://docs.python.org/3/library/typing.html#typing.Never). [assert_never](https://docs.python.org/3/library/typing.html#typing.assert_never) is a function that only accepts the `Never` type, therefore this would fail if we ever added any more values to the enum. 

You can always use unit tests to test a dictionary's exhaustiveness, but static type checkers tend to get you the feedback a lot quicker and without extra tests. 


### Dictionary dispatch pattern
Dictionary dispatch pattern is a special case of the dictionary mapping pattern. It's essentially a form of polymorphism, instead of a value mapping to a value, we map a value to a function. For example:

```py
def handle_save(data, filename="backup.txt",  *_, **_):
    return f"Saving '{data}' to {filename}"

def handle_delete(item_id):
    return f"Deleting record: {item_id}"

def handle_error(*_, **__):
    return f"Invalid command."

dispatch_table = {
    "save": handle_save,
    "delete": lambda item_id, *_, **__: handle_delete(item_id),
}

def execute_command(command, *args, **kwargs):
    action = dispatch_table.get(command, handle_error)
    return action(*args, **kwargs)
```

The benefit is for developers to create handler function adhering to the same abstraction. 

The disadvantage is that forcing the same abstraction can be too restrictive .

Notice how we use a lot of `*_`, `**__`, this is because every function now needs to be able to handle the same set of arguments. Sometimes `partial` or `lambda` is even used to massage the arguments. 

Another issue I have is a constant indirection, when we read this type of code, we are doing a lot of 'hops'. 

Reading `execute_command` then hopping to the dispatch table, then we have to hop to individual functions. If `partial`s and `lambdas` are involved then that's even more hops. 

Let's contrast that to the `if` ... `else` ... implementation: 

```py
def handle_save(data, filename="backup.txt", *_, **_):
    return f"Saving '{data}' to {filename}"

def handle_delete(item_id):
    return f"Deleting record: {item_id}"


def execute_command(command, item_id, data, filename):
    if command == "save":
        return handle_save(data, filename)
    
    elif command == "delete":
        return handle_delete(item_id)
            
    else:
        return "Invalid command."
```
We no longer have to coerce all the function arguments into the same format in this case, allowing us to call the function with concise arguments. 

Of course not all examples are this extreme but I see a lot of unnecessary complexity like this day to day. Some of which started with simple functions and grew out of control. 

## The Point
Having gotten so far, I will admit that this all sounds very ranty—apologies. 

I want to stress that where there are multiple ways to do something, the common approach is not always the best approach. When we find ourselves in these situations, don't just look for the 'Pythonic' implementation, and definitely don't go counting whitespace. Instead, think critically about what you’re trying to achieve. If you still can't decide, go with the simplest, low-abstraction approach; you can always revisit it another day