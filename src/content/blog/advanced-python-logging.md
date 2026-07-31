---
title: "Some Advanced Python Topics - Logging"
description: "Quick notes on logging beyond basicConfig: loggers, handlers, formatters, filters, and queue handling."
pubDate: 2026-07-06
heroImage: "../../assets/python-advanced-logging-hero.png"
---

As I turn more of my work to Python because of AI/ML, coming from Java distributed-system roles, I want to note down some advanced Python topics that I find useful. This first one is about logging. In my previous Java roles, production logging often came with the framework. In Python, the standard library has most of the same pieces.

This is not an introduction to logging. It assumes we already use `logger.info()` and know the normal log levels. I just want to go through the main components in this post.

The mental model is simple:

`Logger -> Handler -> Filter -> Formatter -> output`

A logger creates a log record. A handler decides where it goes. A filter can drop it or add information to it. A formatter turns it into the final text or JSON. The [Python Logging Cookbook](https://docs.python.org/3/howto/logging-cookbook.html) goes much deeper, but these are the parts I use most.

![How a Python LogRecord flows from application code to one or more handlers, then through filters and formatters to their outputs.](/python-logging-flow.svg)

_One logger can send the same `LogRecord` to more than one handler. Each handler can have a different level, filter, formatter, and output._

## Table of Contents

- [The main logging components](#the-main-logging-components)
  - [Logger](#logger)
  - [Handler](#handler)
  - [Formatter](#formatter)
  - [Filter](#filter)
- [Useful patterns](#useful-patterns)
  - [Configuration with `dictConfig`](#configuration-with-dictconfig)
  - [`QueueHandler`](#queuehandler)

## The main logging components

### Logger

The logger is what our application code talks to. I normally create one logger per module:

```python
# myapp/payments.py
import logging

logger = logging.getLogger(__name__)


def charge(order_id: str) -> None:
    logger.info("starting charge", extra={"order_id": order_id})
```

`__name__` gives us names such as `myapp.payments` instead of every line coming from the root logger. That becomes useful when searching logs, and it also gives us a hierarchy: `myapp` is the parent of `myapp.payments`.

One thing that confused me at first: the logger does not really decide where the message is written. It creates the record and passes it up to handlers. So I use `logging.getLogger(__name__)` freely in application code, but configure handlers only once when the app starts.

### Handler

A handler decides where the record goes. The most common ones are `StreamHandler` for the console, `FileHandler` for a file, and `RotatingFileHandler` for a file that should not grow forever.

One logger can use more than one handler. For example, this writes all debug information to a rotating file, but only prints info and above to the console:

```python
import logging
from logging.handlers import RotatingFileHandler

root = logging.getLogger()
root.setLevel(logging.DEBUG)

console = logging.StreamHandler()
console.setLevel(logging.INFO)

debug_file = RotatingFileHandler(
    "app.log", maxBytes=10_000_000, backupCount=3
)
debug_file.setLevel(logging.DEBUG)

root.addHandler(console)
root.addHandler(debug_file)
```

Both the logger level and handler level matter. If the root logger is `INFO`, a `DEBUG` record never reaches the file handler, even when that handler is set to `DEBUG`.

For a containerized service, I usually prefer writing to stdout and letting the platform collect logs. But file handlers are still handy for a local worker or a simple long-running script. If a named logger gets its own handler, set `logger.propagate = False` or it may also pass the record to the root handler and show up twice.

### Formatter

A formatter controls what a handler writes. Here is a useful human-readable one:

```python
formatter = logging.Formatter(
    "%(asctime)s %(levelname)-8s %(name)s "
    "%(filename)s:%(lineno)d %(message)s"
)

console.setFormatter(formatter)
debug_file.setFormatter(formatter)
```

I like having the logger name and line number when debugging locally. In production, I usually prefer JSON because log platforms can search its fields directly.

```python
import json
import logging
from datetime import datetime, timezone


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        event = {
            "timestamp": datetime.fromtimestamp(record.created, timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }

        for field in ("request_id", "order_id", "model_name", "retry_count"):
            value = getattr(record, field, None)
            if value is not None:
                event[field] = value

        if record.exc_info:
            event["exception"] = self.formatException(record.exc_info)

        return json.dumps(event, default=str)
```

The important thing is not to include every possible field in every log event. I normally keep a small, predictable set of fields that I know I will query: request ID, order ID, model name, retry count, and so on.

### Filter

Log levels already filter the normal severity levels. A filter is for extra logic: it can drop a record, or it can add information to it.

The most useful filter I have used adds a request ID to every log line. Without it, a deep function has no idea which web request it belongs to. We could pass `request_id` through every function, but that gets noisy quickly.

```python
# myapp/logging_setup.py
import logging
from contextvars import ContextVar

request_id = ContextVar("request_id", default="-")


class RequestContextFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id.get()
        return True
```

At the request boundary, set it and always reset it:

```python
from uuid import uuid4

from myapp.logging_setup import request_id


async def handle_request(request):
    incoming_request_id = request.headers.get("x-request-id")
    token = request_id.set(incoming_request_id or uuid4().hex)
    try:
        return await run_business_logic(request)
    finally:
        request_id.reset(token)
```

Then a deep function can continue to log normally:

```python
logger.info("charge accepted", extra={"order_id": order_id})
```

`ContextVar` is Python's built-in task-local storage. Think of it as a value attached to the current request's execution context, instead of a normal global variable. This matters in an async service: request A can set its ID, pause at `await`, request B can run on the same thread, and then request A can continue. A module-level variable would have been overwritten by request B. `ContextVar` keeps the value for each `asyncio` task (and also works with threads), so the filter sees the right request ID. A filter can be attached to either a logger or a handler; I attach this one to the handler, so it applies to every logger that reaches that handler:

```python
console.addFilter(RequestContextFilter())
```

`extra` is still useful for event-specific fields such as `order_id`. The filter is for the context we want on every event. I also try to keep that context boring: IDs are useful; access tokens, passwords, and raw request bodies should not be in logs.

## Useful patterns

### Configuration with `dictConfig`

The examples above construct objects in Python so we can see the pieces. For an application, I prefer connecting them once with `dictConfig`. It keeps the setup in one place and avoids every module adding its own handler.

```python
# myapp/logging_setup.py
import logging.config

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "filters": {
        "request_context": {"()": "myapp.logging_setup.RequestContextFilter"},
    },
    "formatters": {
        "default": {
            "format": "%(asctime)s %(levelname)s %(name)s "
            "request_id=%(request_id)s %(message)s",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "stream": "ext://sys.stdout",
            "level": "INFO",
            "formatter": "default",
            "filters": ["request_context"],
        },
    },
    "root": {"level": "INFO", "handlers": ["console"]},
}


def configure_logging() -> None:
    logging.config.dictConfig(LOGGING)
```

Call `configure_logging()` once in `main`, before the application starts. `disable_existing_loggers=False` is a small but useful default: it avoids accidentally hiding framework or client-library logs.

If we want JSON in this config, we can add `"json": {"()": "myapp.logging_setup.JsonFormatter"}` under `formatters` and point the handler's `formatter` to `json`.

### `QueueHandler`

Some handlers can block. Writing one line to stdout is usually cheap; sending a log to a remote service or a slow file system is not. `QueueHandler` lets the request put a record on a queue, while a background listener handles the actual output.

`QueueHandler` and `QueueListener` work as a pair. The `QueueHandler` runs in the request thread and only enqueues the `LogRecord`. `QueueListener` starts a background thread, takes records from that queue, and sends them to ordinary handlers such as `StreamHandler`, `FileHandler`, or an HTTP handler. In the example below, `sink` is the ordinary handler that actually writes the log.

```python
import logging
from logging.handlers import QueueHandler, QueueListener
from queue import SimpleQueue


def start_queued_logging() -> QueueListener:
    events = SimpleQueue()
    sink = logging.StreamHandler()
    sink.setFormatter(
        logging.Formatter(
            "%(asctime)s %(levelname)s request_id=%(request_id)s %(message)s"
        )
    )

    # Starts a background thread that drains `events` and calls `sink`.
    listener = QueueListener(events, sink, respect_handler_level=True)
    listener.start()

    root = logging.getLogger()
    root.handlers.clear()
    root.setLevel(logging.INFO)

    queue_handler = QueueHandler(events)
    queue_handler.addFilter(RequestContextFilter())
    root.addHandler(queue_handler)
    return listener
```

This helper replaces the root handlers, so treat it as an alternative to the earlier `dictConfig` setup, not as a second setup to paste after it.

Keep the returned listener and call `listener.stop()` in the application's shutdown hook:

```python
def main() -> None:
    listener = start_queued_logging()
    try:
        run_server()
    finally:
        listener.stop()
```

The `finally` block lets the listener finish the records already on the queue before the process exits. In FastAPI or another framework, the same call belongs in its lifespan or shutdown hook.

`RequestContextFilter` (from the example above) reads `request_id` from the current `ContextVar` and adds it to the `LogRecord`. It has to run on `queue_handler`, before the record enters the queue. The listener runs in another thread, which does not have the original request's `ContextVar`; adding the filter to `sink` would only show the default `-`.

For multiple worker processes, do not let all of them write directly to one normal file handler. Use a `multiprocessing.Queue` and one listener process instead, like the [multi-process example in the cookbook](https://docs.python.org/3/howto/logging-cookbook.html#logging-to-a-single-file-from-multiple-processes).
