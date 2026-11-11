# fork_sample_call_fn.py

```
import os

def my_func(x, y):
    print(f"Child running my_func with {x=} and {y=}")
    return x * y

_test = 0

pid = os.fork()


if pid == 0:
    # This is the child process
    my_func(10, 20)

    print(f"In child process: {id(my_func)}")
    os._exit(0)       # terminate child cleanly
else:
    # This is the parent process
    print(f"Parent: child PID is {pid}")
    # os.waitpid(pid, 0)

    print(f"In parent process: {id(my_func)}")

```## tasks.py
```python
import time

def cpu_task():
    s = time.time()
    for _ in range(50_000_000):
        pass
    print("Process finished in", time.time() - s)
```## async_sample.py
```python
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

asyncio.run(main(), debug=True)
```## fork_sample_bidirect.py
```python
import os
import time
# Create two pipes
pr, pw = os.pipe()  # parent writes, child reads
cr, cw = os.pipe()  # child writes, parent reads

pid = os.fork()

if pid == 0:
    # --- Child ---
    os.close(pw)
    os.close(cr)

    msg = os.read(pr, 1024) # Will block to wait compuate
    print("Child got:", msg.decode())

    reply = b"Message received!"
    os.write(cw, reply)

    os.close(pr)
    os.close(cw)
else:
    # --- Parent ---
    os.close(pr)
    os.close(cw)

    print("[Parent] Started, waiting 2 seconds before sending message...")
    time.sleep(2)


    os.write(pw, b"Hello child!")
    reply = os.read(cr, 1024)
    print("Parent got:", reply.decode())

    os.close(pw)
    os.close(cr)

```## process_sample.py
```python
import os

pid = os.fork()

if pid == 0:
    print("Child process")
else:
    print("Parent process, child PID:", pid)
```## fork_sample.py
```python
import os

# Create a pipe: r for reading, w for writing
r, w = os.pipe()

pid = os.fork()


if pid == 0:
    # --- Child process ---
    os.close(r)  # child doesn't read

    message = b"Hello from child!"
    os.write(w, message)
    os.close(w)  # important: tell parent we are done writing
else:
    # --- Parent process ---
    os.close(w)  # parent doesn't write

    # Read data sent by child
    data = os.read(r, 1024)
    print("Parent received:", data.decode())

    os.close(r)
```
