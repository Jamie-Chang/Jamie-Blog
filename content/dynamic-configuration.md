Title: Dynamic config part 1: Pydantic and file watchers
Date: 2025-08-26
Category: Blog

Feature flags or dynamic configuration is something that I find very useful, however I've never had the chance to use them. This is for a lack of options [launchdarkly](https://launchdarkly.com/), [flagsmith](https://www.flagsmith.com/) and [unleash](https://www.getunleash.io/) to name a few. 

SaaS options can be amazing with a full array of features, but I would love a simple solution to start with that can be set up in your own cluster.

I don't have a full solution in my head, instead I'm going to try and explore the space and work towards a solution over the next few posts.

## pydantic-settings

[pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) would probably be the go to solution in Python for config management. Example configuration will look something like this:

```py
class Settings(BaseSettings):
    setting1: str
    setting2: str

    model_config = {"toml_file": Path("config.toml")}

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        *args: object,
        **kwargs: object,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        return (TomlConfigSettingsSource(settings_cls),)


settings = Settings()
```

We have a pydantic model with the fields `"setting1"` and `"setting2"`. There's a `model_config` that contains metadata which acts as the "settings of settings". The `settings_customise_sources` method defines the sources used to populate the and in what order. 

I do have my misgivings how this works, mainly around the way the sources and tied to the object model. Something more functional would be clearer:

```py
settings = load(
    Settings,
    toml_loader("config.toml"),
    env_loader(".env"),
    ...,
)
```
But that's a topic for another day. 

## Config from file
Environment variables are undoubtedly the easiest way to configure an application. However there is no good method to update the envvar during a process' execution and have process reload it. 

Another approach is to load configuration from network resources such as another service or databases. This is likely how existing SaaS solutions work. This is a viable choice that offers a ton of flexibility, the potential operational overhead is worrying, for example, what happens if we fail to load the config from the remote? We likely need to handle many different failure nodes in our application.

That's why I've chosen to configure the app a file as in the above example. Which is simple and more reliable than the network method. The file content can easily be changed by another process or manually. Likewise it can easily be viewed for debugging purposes.

Note that I've chosen to use toml in the above example, but pydantic-settings includes loaders for almost all [formats](https://docs.pydantic.dev/latest/concepts/pydantic_settings/#other-settings-source). My personal preference is toml for small amount of config and yaml for potentially large config files. 

## Reloading the file dynamically
That leaves us with the biggest question, how will we reload the file dynamically? 

Luckily this is a question that already has well established answers: file watchers. File watchers are often used for local development to allow for dynamic reloading of servers. [inotify](https://man7.org/linux/man-pages/man7/inotify.7.html) is one such exapmle on linux where a lot of server code is being run.

There are already many packages that make use of inotify, I've chosen [watchfiles](https://watchfiles.helpmanual.io/) for its broad compatibility across multiple platforms and generally great performance.

The api is incredibly straightforward:

```py
async for _ in awatch(CONFIG_PATH):
    ...  # handle the file change
```

We enter a new iteration every time there's file change, during this iteration we can reload the settings. There are sync apis available as well but I'll stick to async for now. 

## Putting things together
I've added everything together in a FastAPI service. The service has a single endpoint to fetch the current settings values. Django and flask services can also use something similar to this, which I may cover in the near future.

```py
# /// script
# requires-python = ">=3.13"
# dependencies = [
#     "fastapi",
#     "pydantic-settings",
#     "uvicorn",
#     "watchfiles",
# ]
# ///
import asyncio
from contextlib import asynccontextmanager
from pathlib import Path
from typing import AsyncIterator, Self

import uvicorn
from fastapi import FastAPI
from pydantic_settings import (
    BaseSettings,
    PydanticBaseSettingsSource,
    TomlConfigSettingsSource,
)
from watchfiles import awatch


CONFIG_PATH = Path("config.toml")


class Settings(BaseSettings):
    setting1: str
    setting2: str

    model_config = {"toml_file": CONFIG_PATH}

    @classmethod
    def load(cls) -> Self:
        return cls()  # type: ignore

    def reload(self) -> None:
        self.__init__()  # type: ignore

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        *args,
        **kwargs,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        return (TomlConfigSettingsSource(settings_cls),)


settings = Settings.load()


async def auto_reload_settings():
    async for _ in awatch(CONFIG_PATH):
        settings.reload()


@asynccontextmanager
async def lifespan(_: FastAPI) -> AsyncIterator[None]:
    async with asyncio.TaskGroup() as tg:
        task = tg.create_task(auto_reload_settings())
        try:
            yield
        finally:
            task.cancel()



app = FastAPI(lifespan=lifespan)


@app.get("/settings")
def get_settings() -> Settings:
    return settings


if __name__ == "__main__":
    uvicorn.run(app)

```

### A few notes
#### Reloading and loading the settings
The documentation for inplace reloading can be found [here](https://docs.pydantic.dev/latest/concepts/pydantic_settings/#in-place-reloading). The direct invocation of `__init__` is a little odd so I've decided to encapsulate it as `reload` but otherwise this is just following the official documentation.

#### Running a background asyncio task
[lifespan](https://fastapi.tiangolo.com/advanced/events/#use-case) is the best way to add startup and teardown logic to a fastapi app. 

[asyncio.TaskGroup](https://docs.python.org/3/library/asyncio-task.html#task-groups) is used here to handle the task future and avoid not await warnings. 

## Next steps
I think what I've put together here is a very simple solution, but simple solutions can work more reliably. 

This is of course only the first step in hopefully a series of posts.In the near future I'll explore:

- Connecting the code with the infrastructure (e.g. kubernetes) we're running the code on. 
- Solving the observability challenge that comes with dynamically changing configuration.
