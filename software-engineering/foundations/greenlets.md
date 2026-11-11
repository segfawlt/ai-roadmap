# `greenlet`

``` mermaid
        sequenceDiagram
    participant Main as main greenlet
    participant G as greenlet g (foo)
    Main->>G: g.switch() — start executing foo
    G->>G: print("foo 1")
    G->>Main: main.switch("hello") — yield "hello" to main
    Main->>Main: receives "hello" from g.switch()
    Main->>Main: print("ABC")
    Main->>G: g.switch() — resume foo
    G->>G: print("foo 2")
    G->>Main: foo returns "123" — g is now dead
    Main->>Main: g.dead == True
```

``` python
from greenlet import greenlet

def foo():
    print("foo 1")
    main.switch("hello") # So switch back to main and result "Hello"
    print("foo 2")
    return "123"

main = greenlet.getcurrent()

# Create a new greenlet g that will run the foo function.
g = greenlet(foo)
# g exists but not run


# Then `g.switch()` start executing greenlet >> run foo()
print(g.switch())      # → "hello"



print("ABC")
print(g.dead)

g.switch()

print(g.dead)
```

- Print “foo 1”.

- Call main.switch(“hello”) → this yields control back to the main greenlet, passing “hello” as the return value of the g.switch() call that started this greenlet.

  - You can pause and resume it explicitly using .switch()

- After the main greenlet switches back to g, “foo 2” is printed.

``` python
a = g.switch()
```

``` python
a
```

``` python
```

## 4. Why this is useful

Even in a single thread, greenlets let you:

Break up blocking code into chunks.

Pause execution at a point and return a value to the caller.

Resume exactly where you left off later.

Integrate synchronous-looking code with asynchronous frameworks (like asyncio) without threads.

------------------------------------------------------------------------

## Shared data

## 1. **Shared data is normal Python data**

All greenlets in the same thread can access the same variables, objects, lists, dictionaries, etc.

``` python
from greenlet import greenlet

shared_list = []

def task1():
    for i in range(3):
        shared_list.append(f"task1-{i}")
        main.switch()

def task2():
    for i in range(3):
        shared_list.append(f"task2-{i}")
        main.switch()

main = greenlet.getcurrent()
g1 = greenlet(task1)
g2 = greenlet(task2)

while not g1.dead or not g2.dead:
    if not g1.dead: g1.switch()
    if not g2.dead: g2.switch()

print(shared_list)
```

Output (order may vary, depending on switching):

```
['task1-0', 'task2-0', 'task1-1', 'task2-1', 'task1-2', 'task2-2']
```

Key point: Both greenlets **see the same `shared_list`**, because it’s in the same memory space.

------------------------------------------------------------------------

## 2. **No thread safety concerns (but watch out)**

- Since greenlets run **in a single thread**, you **don’t need locks** like in multithreading.

- No two greenlets are executing Python bytecode at the same time; one greenlet runs until it calls `.switch()`.

- However, **if you switch in the middle of modifying a mutable object**, you could get inconsistent state **if you don’t control switching points carefully**.

Example pitfall:

``` python
data = []

def g1_func():
    for i in range(5):
        data.append(i)
        main.switch()  # switching mid-operation

def g2_func():
    for i in range(5, 10):
        data.append(i)
        main.switch()

# switching back and forth might interleave operations in unexpected ways
```

- It’s not a “race condition” in the OS-thread sense, but **logical races** can occur if you depend on sequential updates.

------------------------------------------------------------------------

## 3. **Local state per greenlet**

- Each greenlet has its own stack and local variables.

- **Local variables inside a greenlet are isolated**, just like function locals in Python.

``` python
def foo():
    local_var = 123
    main.switch()
    print(local_var)  # still 123 in this greenlet
```

- Only **variables outside the function**, or objects you explicitly pass/share, are shared.

------------------------------------------------------------------------

## 4. **Summary**

| Aspect                | Greenlet behavior              |
|-----------------------|--------------------------------|
| Memory                | Shared (same thread)           |
| Mutable objects       | Shared between greenlets       |
| Local variables       | Private to each greenlet       |
| Thread safety / locks | Not needed (single thread)     |
| Race conditions       | Only logical / ordering issues |

``` python
data = []

def g1_func():
    for i in range(5):
        data.append(i)
        main.switch()  # switching mid-operation

def g2_func():
    for i in range(5, 10):
        data.append(i)
        main.switch()

main = greenlet.getcurrent()
g1 = greenlet(g1_func)
g2 = greenlet(g2_func)

while not g1.dead or not g2.dead:
    if not g1.dead: g1.switch()
    if not g2.dead: g2.switch()
```

------------------------------------------------------------------------

## Print frame

``` python
from greenlet import greenlet
import inspect

def foo():
    print("foo 1")
    main.switch()
    print("foo 2")
    main.switch()

main = greenlet.getcurrent()
g = greenlet(foo)

# Start greenlet
g.switch()

# Inspect last executed frame
frame = g.gr_frame
if frame is not None:
    lineno = frame.f_lineno
    filename = frame.f_code.co_filename
    funcname = frame.f_code.co_name
    print(f"Greenlet stopped at {filename}, function {funcname}, line {lineno}")
```

## Apply in `AsyncEngine`, `AsyncSession`

Async methods like:

``` python
await engine.connect()
await session.execute(...)
await session.commit()
```

DO NOT directly call async database drivers, Instead, they:

1.  Create a greenlet

2.  Run the synchronous engine inside it

3.  “Suspend” and “resume” using greenlet switches

4.  Return an awaitable back to Python’s asyncio loop

``` python
import inspect
import asyncio
from greenlet import greenlet


# Greenlet utility, similar to SQLAlchemy's `greenlet_spawn`
async def greenlet_spawn(func, *args, **kwargs):
    loop = asyncio.get_running_loop()
    fut = loop.create_future()

    def run_in_greenlet():
        try:
            result = func(*args, **kwargs)
            loop.call_soon_threadsafe(fut.set_result, result)
        except BaseException as e:
            loop.call_soon_threadsafe(fut.set_exception, e)

    g = greenlet(run_in_greenlet)
    g.switch()
    return await fut


class AuthUserPassServer:
    def __init__(self, fn_check_userpass):
        assert fn_check_userpass, "Must set fn_check_userpass"
        self.fn = fn_check_userpass

    # -----------------------------------------
    # 1) Synchronous API
    # -----------------------------------------
    def authen_user(self, username, password):
        """Synchronous wrapper that never blocks the event loop."""
        if inspect.iscoroutinefunction(self.fn):
            # Sync API calls async API (LSP preserved)
            return asyncio.run(
                self.async_authen_user(username, password)
            )
        else:
            # Direct sync call
            return self.fn(username, password)

    # -----------------------------------------
    # 2) Asynchronous API
    # -----------------------------------------
    async def async_authen_user(self, username, password):
        """Async API that supports both sync and async functions."""

        if inspect.iscoroutinefunction(self.fn):
            # Case A: fn is async → run its await in a greenlet
            return await greenlet_spawn(
                lambda: asyncio.run(self.fn(username, password))
            )

        else:
            # Case B: fn is sync → run sync fn in greenlet
            return await greenlet_spawn(
                self.fn, username, password
            )
```

## Simple `greenlet` application

``` python
import asyncio
from greenlet import greenlet


async def greenlet_spawn(fn, *args, **kwargs):
    loop = asyncio.get_running_loop()
    future = loop.create_future()
    def fn_run():
        try:
            result = fn(*args, **kwargs) # fn is a function (sync)
            loop.call_soon_threadsafe(future.set_result, result)
        except Exception as e:
            loop.call_soon_threadsafe(future.set_exception, e)

    g = greenlet(fn_run)
    g.switch() # Now call fn_run()
    return await future


def p():
    print("Print something")
    return "Conco"

async def p_async():
    print("Print something 2")
    return "Conco 2"


# Same things
print(await greenlet_spawn(p))

print(await p_async())
```

```
Print something
Conco
Print something 2
Conco 2
```

``` python
from _asyncio import _get_running_loop
```

``` python
_get_running_loop()
```

```
<_UnixSelectorEventLoop running=True closed=False debug=False>
```
