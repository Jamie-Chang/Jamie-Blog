Title: Writing a SQS task framework from scratch
Date: 2026-01-18
Category: Blog
Tags: Python, Asyncio, SQS

Recently I've been working on framework to run LLM tasks using AWS's excellent [SQS](https://aws.amazon.com/sqs/). And I made the decision to write my own task framework/library as opposed to using a pre-exiting framework. I thought this would be a great opportunity to discuss the considerations and levels of abstractions involved when coding a framework.

### Why SQS? 
Having come from using traditional message queues like RabbitMQ, I can't help but compare SQS with RabbitMQ. The thing is, whilst they can be interchangeable. I think they are actually built with different use cases in mind.

RabbitMQ is for processing a large volume of messages, the messages are strictly processed in FIFO order. SQS has more flexibility around when a message is processed and how long each task takes. So RabbitMQ is good for handling events, but SQS is better for tasks like LLM calls.

Additionally, SQS uses http to communicate between the server and the client which is easier to monitor and is easier to setup if you already have an AWS cluster.

### Why not use ...? 
Okay so, the first question I will answer you is why not a pre-existing framework? 

When we talk about task framework in Python, there is a general consensus. The most popular choice is [Celery](https://docs.celeryq.dev/en/stable/getting-started/introduction.html), a framework that can handle pretty much any handle workload in any broker (be it SQS or otherwise). If you need a UI and compose tasks in a DAG (Directed Acyclic Graph) then you should use [Airflow](https://airflow.apache.org/). If you like new technology and talking about orchestration then use something like [temporal](https://temporal.io/). The list really goes on and on. 

I haven't used a lot of managed task solutions like temporal or [prefect](https://www.prefect.io/) which admittedly is something I'd love to try out.

As for why I don't use something like celery. Celery is a framework that tries to work with everything including all brokers. That is only true to an extent as it was originally designed to work with RabbitMQ. Generally the support is also pretty good for Redis but not so good otherwise. A lot of broker specific features are either not supported or not documented.

Another issue is a lack of asyncio support. For my use case asyncio is great, no need for multiple threads or processes. Celery again tries very hard to be general, it supports the concurrency paradigm most likely to support a user's code without a lot of input from the user. But in my case, I already have an opinion of how I want my code to run, and it just doesn't make sense to use it.

Finally, my general complaint of celery is that it requires a lot of configuration for a production use case. Which can cause quite a bit of a headache. 

## Creating my abstractions
In my experience writing a good framework is all about how good the abstractions are. Abstractions can come with a cost. Whilst Celery create abstractions that allow task compositions and provides support for many brokers, but it trades broker specific features and increases in configuration complexity.

Therefore a good abstraction is about making useful trade-offs. 


### Writing the code
Before we start with abstractions, it's a good idea to first write some code to see what the it currently looks like with no abstraction.

Here I ask a LLM to provide an example:

```py
import asyncio
from aiobotocore.session import get_session

# Replace with your actual Queue URL
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456789012/my-queue"

async def consume_messages():
    session = get_session()
    
    # create_client is an async context manager that handles connection cleanup
    async with session.create_client('sqs', region_name='us-east-1') as client:
        print(f"Listening for messages on {QUEUE_URL}...")
        
        while True:
            try:
                # receive_message calls the SQS API
                # WaitTimeSeconds=20 enables Long Polling (waits for messages to arrive)
                response = await client.receive_message(
                    QueueUrl=QUEUE_URL,
                    MaxNumberOfMessages=10,
                    WaitTimeSeconds=20,
                    VisibilityTimeout=60,
                )

                # Check if 'Messages' key exists in response
                if 'Messages' in response:
                    for msg in response['Messages']:
                        # 1. Process the message
                        print(f"Received body: {msg['Body']}")

                        # 2. Delete the message so it isn't processed again
                        await client.delete_message(
                            QueueUrl=QUEUE_URL,
                            ReceiptHandle=msg['ReceiptHandle']
                        )
                        print("Message deleted.")
                else:
                    print("No messages received in this poll.")

            except Exception as e:
                print(f"Error encountered: {e}")
                await asyncio.sleep(5)  # Backoff on error
```

### Identifying Trade-offs
Next we look at things we want to change and thus the trade-offs.

Let's start with deleting the message, the purpose of deleting the message is to signal that the message is complete and shouldn't be processed again. I personally don't like the naming of this, in the context of message queues this is usually called `ack`. 

Something like:
```py
client.ack(message)
```

Next, let's look at some of the attributes we can abstract:

- `VisibilityTimeout`: is the amount of time the message can be processed for before going back on the queue, it is also how we can delay the message for when adding a message to the queue. We can make this clearer by calling it keep-alive and delay respectively.
- `MaxNumberOfMessages`: is a useful feature of SQS to fetch a batch of messages at once to increase throughput and reduce the cloud bill. But it may not be a good fit for my use case handling LLM requests as they don't work well in batches
- `WaitTimeSeconds` enables long polling which reduces the number of requests and hence the cloud bill. This is a good idea to keep on. And there's no real need for us to turn it off. 

### Creating the client
Now that the trade offs are identified, we will start to create the client code, and this is what I've come up with:

```py
@dataclass
class QueueClient:
    client: SQSClient
    queue: str
    poll_interval: int = 20

    async def consume(self, keep_alive: int = 30) -> AsyncIterator[MessageTypeDef]:
        while True:
            messages = await self.client.receive_message(
                QueueUrl=self.queue,
                MaxNumberOfMessages=1,
                WaitTimeSeconds=self.poll_interval,
                VisibilityTimeout=keep_alive,
            )
            match messages:
                case {"Messages": [message]}:
                    yield message
                case _:
                    continue

    async def ack(self, message: MessageTypeDef) -> None:
        await self.client.delete_message(
            QueueUrl=self.queue,
            ReceiptHandle=message["ReceiptHandle"],
        )
```

In order to pass the messages to the user, I've represented the stream of messages as an AsyncIterator. Iterators are truly one of Python's best features, and here it fits particularly well as are trying to expose the messages to the user without making may assumptions about how to user wants to consume it.

Consuming messages then is as simple as running an async for loop:

```py
async def main():
    async for message in client.consume():
        await handle(message)
        await client.ack(message)
```


### Error Handling

A good task framework should provide utilities to handle errors well, in our scenario we simply terminate if we encounter an error. This could be enough provided that we automatically restart our process, but generally this is not the expectation.

The standard terminology for this is `nack` the opposite of `ack`. In SQS terms this is simply [`change_message_visibility`](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sqs/client/change_message_visibility.html#SQS.Client.change_message_visibility) or `delete_message` depending whether we want to retry.


So inside `QueueClient` goes:

```py
@dataclass
class QueueClient:
    ...

    async def nack(self, message: MessageTypeDef, retry_timeout: int | None = None) -> None:
        if retry_timeout is None:
            await self.ack(message)
            return

        await self.client.change_message_visibility(
            QueueUrl=self.queue,
            ReceiptHandle=message["ReceiptHandle"],
            VisibilityTimeout=retry_timeout,
        )
```



Consuming messages with error handling is:

```py
async def main():
    async for message in client.consume():
        try:
            await handle(message)
        except Exception:
            await client.nack(message)
            logger.error(...)
        else:
            await client.ack(message)
```

This may look a little ugly right now, but we want to avoid abstracting things prematurely, so I've left this for later.

### Extending keep alive
One thing I wanted to do over celery is to manage the timeout of the message. Celery is configured with a static value which doesn't work well for a something like LLM queries.

Extending the keep-alive is essentially the same operation as retrying: `change_message_visibility` in one case we set the timeout so that we can retry after a period of time, in the other case we extend the timeout so that we won't retry too soon. 

In the `QueueClient` goes:
```py
@dataclass
class QueueClient:
    ...

    async def keep_alive(self, duration: int) -> None:
        await self.client.change_message_visibility(
            QueueUrl=self.queue,
            ReceiptHandle=message["ReceiptHandle"],
            VisibilityTimeout=duration,
        )
```

## Abstracting consumption

As it is right now, our `QueueClient` can be tested and used. We know that it will function quite well because we've been careful to make abstraction decisions that improve usability but minimise the cost.

I hinted early that we could do a bit more abstraction. As we have made our `QueueClient` simple, the consumer loop is more complex than it needs to be, requiring the user to understand how to manage the lifecycle of a message. 

We've left this out because there are many different ways the user may want to manage the message's lifecycle, so enforcing one way may restrict the user's options. 

However, we can provide some abstraction with helpful implementation without narrowing the user's options. 

To do this we can use a abstract base class or a [Protocol](https://typing.python.org/en/latest/spec/protocol.html) to define what we want the `LifeCycle` to look like:

```py
class LifeCycle(Protocol):
    """Defines the lifecycle of a single message.

    That is what happens 
        - when a message is processed successfully
        - when a message errors
        - during the message processing
    """
    def __call__(self, message: MessageTypeDef) -> AsyncContextManager[MessageTypeDef]:
        ...
```
We use a `AsyncContextManager` here because it gives us the most flexibility, wrapping the message handler.

So our basic version of message processing would look something like: 

```py
@dataclass
class BasicLifeCycle:
    client: QueueClient

    @asynccontextmanager
    async def __call__(self, message: MessageTypeDef) -> AsyncIterator[MessageTypeDef]:
        try:
            yield message
        except Exception:
            await self.client.nack(message)
            logger.error(...)
        else:
            await self.client.ack(message)

```

We can improve on this with a retry option:


```py
@dataclass
class RetryLifeCycle:
    client: QueueClient
    retry_interval: int

    @asynccontextmanager
    async def __call__(self, message: MessageTypeDef) -> AsyncIterator[MessageTypeDef]:
        try:
            yield message
        except Exception:
            await self.client.nack(message, self.retry_interval)
            logger.error(...)
        else:
            await self.client.ack(message)

```

And adding more complexity for convenience, we can also provide a background task to keep the message alive. This allows us to process the message for a functionally infinite amount of time:


```py
@dataclass
class HeartbeatLifeCycle:
    client: QueueClient
    interval: int

    @asynccontextmanager
    async def auto_keep_alive(self):
        async def _keep_alive():
            while True:
                await asyncio.sleep(self.interval * 0.8)
                await self.client.keep_alive(self.interval)
    
        async with asyncio.TaskGroup() as tg:
            task = tg.create_task(_keep_alive)
            try:
                yield
            finally:
                task.cancel()

    @asynccontextmanager
    async def __call__(self, message: MessageTypeDef) -> AsyncIterator[MessageTypeDef]:
        try:
            async with self.auto_keep_alive():
                yield message
        except Exception:
            await self.client.nack(message)
            logger.error(...)
        else:
            await self.client.ack(message)

```

## Finally putting everything together

Let's compose what a single consumer would look like:

```py
async def main():
    client = QueueClient(...)
    life_cycle = HeartbeatLifeCycle(client, interval=20)
    async for message in client.consume():
        async with life_cycle(message) as message:
            handle(message)  # That's it!

```

Pretty easy right? And should the user want a different lifecycle or to not use it, they are free to do so without any restrictions enforced by the framework. 

The philosophy here is to steer your user to the right direction but ultimately to trust them. A principal I find most frameworks violating. 

### Scaling it
You might wonder if we can scale the task consumption. After all we must be using asyncio for a reason. We needn't worry about coming up with abstractions ourselves as asyncio already comes with [TaskGroup](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup).


```py
async def run(client: QueueClient) -> None:
    life_cycle = HeartbeatLifeCycle(client, interval=20)
    async for message in client.consume():
        async with life_cycle(message) as message:
            handle(message)


async def main(workers: int):
    client = QueueClient(...)
    async with asyncio.TaskGroup() as tg:
        for _ in range(workers):
            tg.create_task(run(client))
```

When there's a good standard library or well known abstraction, this lowers the cost on the user, as we are not creating anything new. 

## Closing Words
Creating the perfect abstraction is no small task, a good place to start is something humble and go from there. But even then it doesn't always work out in your first try. Sometimes we need a lot of iterations to get there.

I see some crazy abstractions that grew ad-hoc over a lengthy amount of time, gaining more and more features but not taking care of the abstraction. This is a common issue with even very popular frameworks, it's okay to recognise an abstraction is not working and start over. Because only then can we work towards something better.

If you're interested in using this library/framework, it's published to pypi as [simple-async-sqs](https://pypi.org/project/simple-async-sqs/). 

