---
title: "Some Advanced Python Topics - Async and Parallel Jobs"
description: "Quick notes on asyncio, ThreadPoolExecutor, and ProcessPoolExecutor, with examples of when to use each."
pubDate: 2026-07-15
heroImage: "../../assets/python-concurrency-hero.png"
---

I was once asked in an interview to explain the difference between `asyncio`, `ThreadPoolExecutor`, and `ProcessPoolExecutor`, and when to use each one. I had used `ThreadPoolExecutor` before, but I was not very familiar with `ProcessPoolExecutor`. Coming from Java distributed-system roles, I mostly thought of executors as thread pools because Java's standard library has no direct equivalent of Python's `ProcessPoolExecutor`.

So I want to write a post to summarize how the three options work and when to use each one.

## Table of Contents

- [Concurrency vs. parallelism](#concurrency-vs-parallelism)
- [asyncio](#asyncio)
- [ThreadPoolExecutor](#threadpoolexecutor)
- [ProcessPoolExecutor](#processpoolexecutor)
- [A note about newer Python versions](#a-note-about-newer-python-versions)

## Concurrency vs. parallelism

The first thing to clarify is that async does not necessarily mean parallel.

**Concurrency** means several jobs can make progress during the same period. While one job waits for the network, another job can run.

**Parallelism** means several jobs are actually executing at the same time, usually on different CPU cores.

I find it easier to start from the work itself:

- If the job mostly **waits for I/O**, use concurrency.
  - If the library already has an async API, use `asyncio`.
  - If the library is synchronous and blocking, use `ThreadPoolExecutor` to move that wait to worker threads.
- If the job mostly **calculates** in pure Python, use `ProcessPoolExecutor` so it can run on multiple CPU cores.

So the first question is not “which one is faster?” It is “is this job mostly waiting, or mostly calculating?”

## asyncio

[`asyncio`](https://docs.python.org/3/library/asyncio.html) normally runs an event loop on one thread. When a task awaits an asynchronous I/O operation, its coroutine pauses and gives control back to the event loop. The next line in that coroutine cannot run yet, but the event loop can run another task while the I/O is in progress. In that sense, `await` is locally paused but globally non-blocking. This is cooperative scheduling: tasks have to give control back to the loop. A synchronous network or database call does not do that, so it blocks the event-loop thread.

```python
import asyncio


async def call_service(name: str) -> str:
    # Imagine an async HTTP or database call here.
    await asyncio.sleep(1)
    return f"{name} finished"


async def main() -> None:
    async with asyncio.TaskGroup() as group:
        user = group.create_task(call_service("user"))
        orders = group.create_task(call_service("orders"))

    print(user.result())
    print(orders.result())


asyncio.run(main())
```

Both calls finish in about one second because they wait concurrently. If I wrote this instead, they would take about two seconds:

```python
user = await call_service("user")
orders = await call_service("orders")
```

The difference is task creation. In the first example, [`TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#task-groups) creates both tasks before waiting for the group to finish. In the second example, there is no `TaskGroup` and no separate task: the first `await` finishes before Python even calls `call_service("orders")`.

Before `TaskGroup` was added in Python 3.11, the common way to run both calls concurrently was `asyncio.gather()`:

```python
async def main() -> None:
    user, orders = await asyncio.gather(
        call_service("user"),
        call_service("orders"),
    )

    print(user)
    print(orders)
```

`gather()` schedules both calls and returns their results in the same order. `TaskGroup` gives the tasks a clearer shared lifecycle: when one task fails, it cancels the remaining tasks and waits for them before leaving the block.

These calls overlap because `asyncio.sleep()` gives control back to the event loop while it waits. If `call_service()` used `time.sleep()`, a synchronous HTTP client, or a long CPU loop instead, it would hold the event-loop thread and prevent the other task from running. Neither `TaskGroup` nor `gather()` can make blocking code non-blocking. `asyncio` works best when the libraries underneath it provide async APIs too.

## ThreadPoolExecutor

Not every HTTP, database, or storage library has an async API. When I need several synchronous I/O operations to overlap, [`ThreadPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#threadpoolexecutor) is the usual option. It is close to Java's `ExecutorService`: we submit normal synchronous functions, and a fixed set of worker threads executes them.

```python
from concurrent.futures import ThreadPoolExecutor
from urllib.request import urlopen


def download(url: str) -> int:
    with urlopen(url, timeout=10) as response:
        return len(response.read())


urls = [
    "https://example.com",
    "https://www.python.org",
    "https://docs.python.org",
]

with ThreadPoolExecutor(max_workers=8) as pool:
    sizes = list(pool.map(download, urls))

print(sizes)
```

Here, `pool.map()` sends each URL to an available worker thread and returns the results in the same order as the input. The `with` block manages the pool's lifetime and waits for the worker threads to finish before cleaning them up.

This works well because each thread spends most of its time waiting for a socket. While one thread waits for the operating system to finish a network operation, another thread can run. Threads also share the same process memory, so passing an object is cheap, although shared mutable state still needs synchronization such as a `Lock`.

CPU-heavy Python code is different because of the GIL, a lock inside the standard CPython interpreter. Within one process, only one thread can hold the GIL and execute Python instructions at a time. If eight threads run the same CPU-heavy Python loop, they mostly take turns instead of using eight CPU cores. The extra scheduling can even make the threaded version slower.

## ProcessPoolExecutor

A thread pool helps when synchronous work is waiting, but it does not solve the GIL problem for a CPU-heavy Python loop. That is where [`ProcessPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#processpoolexecutor) fits. It has almost the same `submit()` and `map()` interface as `ThreadPoolExecutor`, but its workers are separate processes. Each process has its own Python interpreter and GIL, so the functions can run on different CPU cores.

I see process pools less often in everyday application code because most services spend more time waiting on networks and databases than doing heavy computation. They are more relevant in data processing, scientific computing, and other CPU-intensive work.

```python
from concurrent.futures import ProcessPoolExecutor


def count_primes(limit: int) -> int:
    count = 0
    for number in range(2, limit):
        is_prime = all(
            number % divisor
            for divisor in range(2, int(number**0.5) + 1)
        )
        if is_prime:
            count += 1
    return count


def main() -> None:
    limits = [80_000, 85_000, 90_000, 95_000]
    with ProcessPoolExecutor() as pool:
        results = list(pool.map(count_primes, limits))
    print(results)


if __name__ == "__main__":
    main()
```

Here, `pool.map()` sends each limit to a worker process, so the prime-counting jobs can run across multiple CPU cores. The `if __name__ == "__main__"` guard prevents a worker that imports this file from creating another process pool.

The submitted function, its arguments, and its return value also need to be picklable, meaning Python must be able to serialize them for transfer between processes. A lambda, a nested function, an open database connection, or a lock is not a good thing to submit.

Processes have real costs: startup is heavier, each worker has separate memory, and inputs and results normally cross the process boundary through serialization. Sending a huge DataFrame to every task can cost more than the parallel work saves. I would try to send small inputs, return small results, and make each submitted job large enough to justify the overhead.

## A note about newer Python versions

Everything above assumes the normal GIL-enabled CPython build, which is still what most environments use. But “threads cannot run Python in parallel” is no longer universally true. CPython has offered an optional [free-threaded build](https://docs.python.org/3/howto/free-threading-python.html) since Python 3.13. When its GIL is disabled, threads can run Python code on multiple cores. It is still a different build with compatibility and performance tradeoffs, and some extension modules may turn the GIL back on. Unless I know an environment uses the free-threaded build, I still assume the GIL is enabled.

Python 3.14 also added [`InterpreterPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#interpreterpoolexecutor). It uses threads, but each worker has an isolated interpreter with its own GIL, which allows multi-core parallelism. The isolation means mutable objects cannot simply be shared, and calls and results are serialized, so its programming model is closer to a process pool than an ordinary thread pool.

For the normal GIL-enabled CPython runtime, the practical rule is simple: use `asyncio` for async I/O APIs, a thread pool for blocking I/O, and a process pool for CPU-heavy pure Python work.
