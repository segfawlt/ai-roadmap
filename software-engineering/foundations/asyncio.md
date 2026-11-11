# What `asyncio` is ?

- `asyncio` is Python’s asynchronous I/O framework.

- It allows you to write code that looks sequential but actually does many tasks concurrently without using multiple threads or processes.

- It’s single-threaded and usually single-process, but it can manage thousands of I/O-bound tasks efficiently.

- Think of it as a cooperative multitasking system inside one thread.

## How it works

- You write coroutines with `async`` ``def`.

- You use `await` to pause a coroutine when it’s waiting for I/O (network, disk, etc.), letting other coroutines run.

- The event loop schedules tasks efficiently.

**You must run in normal script, not in Jupyter Notebook**

``` python
import asyncio

async def say_after(delay, msg):
    await asyncio.sleep(delay)  # non-blocking sleep
    print(msg)

async def main():
    # run two tasks concurrently
    await asyncio.gather(
        say_after(2, "Hello"),
        say_after(1, "World")
    )

asyncio.run(main())
```

## How `asyncio` compares to threads and processes

| Feature | Threading | Multiprocessing | asyncio |
|----|----|----|----|
| Parallel CPU usage | No (GIL) | Yes | No (single thread) |
| Concurrency type | OS-level (preemptive) | OS-level (parallel) | Cooperative (event-driven) |
| Memory overhead | Low | High | Very low |
| I/O-bound tasks | Good | Works, heavier | Excellent (lightweight) |
| CPU-bound tasks | Inefficient | Excellent | Not suitable |
| Complexity | Moderate (locks) | Higher | Moderate (async/await syntax) |

## Purpose / When to use `asyncio`

- Handle many I/O tasks efficiently without threads or processes. Examples: web servers, network clients, chat apps.

- Avoid thread overhead and locking issues. Threads require careful synchronization; asyncio uses single-threaded cooperative multitasking.

- Scale thousands of concurrent tasks cheaply. Example: a server handling 10,000 client connections can use a single process with asyncio instead of 10,000 threads.

### asyncio Introspection CLI (Python 3.14+)

Python 3.14 adds a built-in tool to inspect running asyncio tasks from the command line:

``` bash
# Show all running asyncio tasks in a process
python -m asyncio ps <PID>

# Output: task table with coroutine names, state, and stack traces
```

This is invaluable for debugging production async applications — you can see which coroutines are running, waiting, or stuck without modifying the application code.

------------------------------------------------------------------------

``` python
import asyncio

async def say_after(delay, msg):
    await asyncio.sleep(delay)  # non-blocking sleep
    print(msg)

async def main():
    # run two tasks concurrently
    await asyncio.gather(
        say_after(2, "Hello"),
        say_after(1, "World")
    )

    return 111

def nguoidingoaipho():
    pass

v = main()
```

```
HHHHHH: <_UnixSelectorEventLoop running=True closed=False debug=False>
HHHHHH: <_UnixSelectorEventLoop running=True closed=False debug=False>
HHHHHH: <_UnixSelectorEventLoop running=True closed=False debug=False>
HHHHHH: <_UnixSelectorEventLoop running=True closed=False debug=False>
HHHHHH: <_UnixSelectorEventLoop running=True closed=False debug=False>
HHHHHH: <_UnixSelectorEventLoop running=True closed=False debug=False>
```

``` python
dir(v)
```

``` python
import inspect

inspect.getcoroutinestate(v)
```

``` python
from asyncio.events import _get_running_loop

_get_running_loop()
```

``` python
import asyncio
asyncio.events._running_loop.loop_pid
```
