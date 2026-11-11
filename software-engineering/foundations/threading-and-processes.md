# Threads vs Processes

``` mermaid
        graph TD
    P[Process] --> PM[Own Memory Space]
    P --> PT[Threads]
    PT --> T1[Thread 1]
    PT --> T2[Thread 2]
    PT --> T3[Thread 3]
    GIL[GIL - Global Interpreter Lock] --> |only one thread runs at a time| PT
    T1 --> |I/O bound - releases GIL| IO[I/O Operations]
    T1 --> |CPU bound - blocked by GIL| CPU[CPU Tasks]
    style GIL fill:#f96,stroke:#333
    style PM fill:#9cf,stroke:#333
```

**Process has “Own memory Space”:** A process is an independent execution environment managed by the operating system, which includes its own distinct memory space. This is correctly shown at the top left of the diagram.

**CPython uses one thread at a time (due to GIL):** The core function of the Global Interpreter Lock (GIL): it acts as a mutex (lock) that ensures only one thread can execute Python bytecode at any given moment, even on multi-core processors.

**Threads are I/O Bound (Common usage)**: The diagram correctly shows that threads are typically used for I/O-bound tasks in Python. When an I/O operation occurs (e.g., waiting for a network response or file read), the thread releases the GIL, allowing other threads to acquire it and run, thus achieving concurrency for these types of tasks.

**CPU Bound tasks struggle with the GIL:** The diagram implies that CPU-bound tasks are constrained by the GIL, as only one thread can actively compute at a time. Trying to use multiple threads for purely CPU-bound tasks in CPython will not achieve true parallelism and can even add overhead from constant context switching.

## Threads (only one GIL shared)

``` python
import threading, time

from tasks import cpu_task

t1 = threading.Thread(target=cpu_task)
t2 = threading.Thread(target=cpu_task)

start = time.time()
t1.start(); t2.start()
t1.join(); t2.join()
print("Total time:", time.time() - start)
```

```
Process finished inProcess finished in 2.0034258365631104
 2.012209892272949
Total time: 2.0227320194244385
```

Expected result:

- Each task takes ~1–1.5 seconds

- Total time ≈ sum of both, e.g. ~2–3 seconds

→ No parallel CPU execution → because one GIL

## Multiprocessing (each process has its own GIL)

``` python
from multiprocessing import Process
import time

from tasks import cpu_task


p1 = Process(target=cpu_task)
p2 = Process(target=cpu_task)

start = time.time()
p1.start(); p2.start()
p1.join(); p2.join()
print("Total time:", time.time() - start)
```

```
Process finished in 1.1095759868621826
Process finished in 1.1105716228485107
Total time: 1.2385430335998535
```

## A more direct explanation

``` python
from multiprocessing import Process
p = Process(target=cpu_task)
p.start()
```

The OS does this:

- fork/spawn → create a new OS-level process

- New process loads a new CPython interpreter

- That interpreter initializes its own GIL

- That process runs your function independently on another CPU core

So by design:

- 1 process → 1 interpreter → 1 GIL

- 4 processes → 4 interpreters → 4 GILs

This is why `multiprocessing` can use multiple CPU cores.

## **Threads (`threading` module)**

- A **thread** is a *lightweight unit of execution* inside a process.

- Multiple threads share the **same memory space** of the process.

- Threads are **cheap to create**, but:

  - In **CPython**, the Global Interpreter Lock (GIL) means **only one thread executes Python bytecode at a time**.

  - Threads are mostly useful for **I/O-bound tasks** (networking, reading files, waiting), not CPU-bound computation.

**Example:**

``` python
import threading

def worker():
    print("Thread is running")

t = threading.Thread(target=worker)
t.start()
t.join()
```

Multiple threads can read/write shared data, but you must be careful with locks to prevent race conditions.

## **Processes (`os.fork()` or `multiprocessing`)**

- A **process** is an independent program execution with **its own memory space**.

- Processes **do not share memory** (unless you use special shared memory or queues).

- Processes are heavier than threads but:

  - They bypass the **GIL**, so multiple processes can run Python code **in parallel on multiple CPU cores**.

  - Useful for **CPU-bound tasks**.

**Example using `multiprocessing`:**

``` python
from multiprocessing import Process

def worker():
    print("Process is running")

p = Process(target=worker)
p.start()
p.join()
```

Each process is fully independent. Changes in one process’s memory don’t affect the other.

------------------------------------------------------------------------

## Key Differences Table

| Feature | Thread (`threading`) | Process (`os.fork` / `multiprocessing`) |
|----|----|----|
| Memory | Shared | Separate |
| Creation overhead | Low | High |
| Parallelism in Python | Limited by GIL (CPU-bound) | True parallelism possible |
| Crash isolation | If one thread crashes, all threads may crash | Independent; crash doesn’t affect others |
| Use case | I/O-bound tasks | CPU-bound tasks |

------------------------------------------------------------------------

### Important Note About `os.fork()`

- `os.fork()` is **low-level**, duplicates the current process **exactly as is**.

- It’s dangerous in multi-threaded programs (like Jupyter) because it copies **threads and resources**, leading to potential deadlocks.

- `multiprocessing` is safer because it handles process creation and communication for you.

## So `multiprocessing` can achieve true parallelism

- By creating separate processes, while threading cannot bypass the GIL for CPU-bound tasks.

``` mermaid
        graph LR
    subgraph Process1[Process 1 - own GIL]
        G1[GIL 1] --> Core1[CPU Core 1]
    end
    subgraph Process2[Process 2 - own GIL]
        G2[GIL 2] --> Core2[CPU Core 2]
    end
    subgraph Process3[Process 3 - own GIL]
        G3[GIL 3] --> Core3[CPU Core 3]
    end
    OS[Operating System] --> Process1
    OS --> Process2
    OS --> Process3
    style Process1 fill:#9cf,stroke:#333
    style Process2 fill:#9f9,stroke:#333
    style Process3 fill:#f9c,stroke:#333
```

## What is Threading

Threading in Python is a technique used to achieve concurrent execution by running **multiple threads**—smaller units of a process—simultaneously or in an overlapping manner.

``` mermaid
        graph TD
    subgraph SingleProcess[Single Python Process]
        GIL[GIL - Global Interpreter Lock]
        T1[Thread 1] -->|acquire lock| GIL
        T2[Thread 2] -->|waiting| GIL
        T3[Thread 3] -->|waiting| GIL
        T4[Thread 4] -->|waiting| GIL
        GIL -->|execute bytecode| CPU[CPU]
    end
    style GIL fill:#f96,stroke:#333
    style SingleProcess fill:#eee,stroke:#999
```

- **Single Process**: All threads (T1, T2, T3, T4) operate within a single Python process, as the diagram correctly shows with the large vertical line encompassing everything.

- **The Global Interpreter Lock (GIL)**: The diagram visually represents the GIL as a single lock mechanism that all threads must contend for to execute Python bytecode.

- **No True Parallelism (for CPU-bound tasks)**: The key takeaway, which the diagram illustrates, is that even with multiple threads available, only one thread can hold the lock and execute code at any given moment. This effectively serializes the execution of CPU-bound threads, preventing them from running in parallel across multiple CPU cores. The small green blocks show threads taking turns executing brief bursts of work

``` mermaid
        graph LR
    subgraph Timeline[Execution Timeline - CPU Bound Threads]
        T1A[T1 runs] --> T2A[T2 runs] --> T1B[T1 runs] --> T2B[T2 runs]
    end
    note["Only one thread runs at a time — no speedup for CPU-bound work"]
    style Timeline fill:#eee,stroke:#999
```

``` python
import threading
import time
import os

def cpu_intensive_task(count_to):
    """A function that performs a CPU-bound operation."""
    while count_to > 0:
        count_to -= 1

def run_single_threaded(iterations, count_target):
    """Runs the task once in the main thread."""
    start_time = time.time()
    for _ in range(iterations):
        cpu_intensive_task(count_target)
    end_time = time.time()
    print(f"--- Single-threaded execution time for {iterations} task(s): {end_time - start_time:.4f} seconds ---")

def run_multi_threaded(num_threads, count_target):
    """Runs the task using multiple threads."""
    start_time = time.time()
    threads = []

    for i in range(num_threads):
        t = threading.Thread(target=cpu_intensive_task, args=(count_target,), name=f"Thread-{i}")
        threads.append(t)
        t.start()

    for t in threads:
        t.join()

    end_time = time.time()
    print(f"--- Multi-threaded ({num_threads} threads) execution time: {end_time - start_time:.4f} seconds ---")

if __name__ == "__main__":
    TARGET_COUNT = 50000000
    NUM_THREADS = 4

    print(f"System has {os.cpu_count()} CPU cores available.")
    run_single_threaded(NUM_THREADS, TARGET_COUNT)
    run_multi_threaded(NUM_THREADS, TARGET_COUNT)

    print("\nObservation: The execution times are very similar, confirming that threads in Python's CPython interpreter cannot run CPU-bound tasks in parallel.")
```

## Thread Lifecycle

``` mermaid
        stateDiagram-v2
    [*] --> New: threading.Thread() created
    New --> Runnable: thread.start()
    Runnable --> Running: GIL acquired / scheduled by OS
    Running --> Runnable: GIL released / preempted
    Running --> Blocked: I/O wait / sleep / lock / join
    Blocked --> Runnable: I/O complete / timeout / lock released
    Running --> Terminated: run() method returns or raises
    Terminated --> [*]
```

**Python Thread Lifecycle States**

- **New (Created)**: A thread is in this state once the threading.Thread object is created but before its `start()` method is called.

- **Runnable (Ready/Alive)**: After `thread.start()` is called, the thread is considered “alive” and moves to the runnable state.

- **Running**: The thread is actively executing its tasks. Due to the GIL in CPython, only one Python thread can truly be in the running state at any given time.

- **Blocked/Waiting/Sleeping**: The thread may temporarily pause for I/O, synchronization, sleeping, or joining.

- **Terminated (Dead)**: The thread reaches this final state when its `run()` method completes. A terminated thread cannot be restarted.

## CPython’s internal threading mechanism

``` mermaid
        graph TD
    subgraph CPython[CPython Interpreter]
        GIL[GIL - Global Interpreter Lock]
        subgraph Threads
            T1[Thread 1<br>start_new_thread C]
            T2[Thread 2<br>start_new_thread C]
            T3[Thread 3<br>start_new_thread C]
        end
        T1 -->|acquire T-state lock| GIL
        T2 -->|wait for T-state lock| GIL
        T3 -->|wait for T-state lock| GIL
        GIL --> BC[Execute Python Bytecode]
    end
    style GIL fill:#f96,stroke:#333
    style CPython fill:#eef,stroke:#669
```

- **Threads**: Separate flows of execution created using `start_new_thread` implemented in C

- **T-state locks**: Internal thread state locks in CPython

- **CPython Implementation**: The GIL is an internal CPython mechanism restricting simultaneous execution of Python bytecode

## `threading.local`

``` python
import threading

from _threading_local import local
data = local()

def worker():
    data.x = threading.current_thread().name
    print(data.x)

threads = [threading.Thread(target=worker) for _ in range(3)]
for t in threads:
    t.start()
```

## Thread Pool

``` python
from multiprocessing.pool import ThreadPool

from threading import current_thread
def work(x):
    print(pool._pool)
    return x*x

pool = ThreadPool(processes=2)
results = pool.map(work, [1, 2, 3, 4, 5, 6, 7, 8])
print(results)
```

``` python
from multiprocessing.pool import ThreadPool

def work(x):
    return x*x

pool = ThreadPool(processes=4)
results = pool.map(work, [1, 2, 3, 4])
print(results)
```

```
[1, 4, 9, 16]
```

## Threads share same global scope

``` python
import threading
import time

count = 0

def try_count(sleep_seconds, name):
    global count
    count += 1
    time.sleep(sleep_seconds)
    print(f"\nIn thread: {name}: count = {count}")


t1 = threading.Thread(target=try_count, args=(1, "T1"))
t2 = threading.Thread(target=try_count, args=(2, "T2"))
t3 = threading.Thread(target=try_count, args=(1, "T3"))

t1.start()
t2.start()
t3.start()
```

```
In thread: T1: count = 3
In thread: T3: count = 3
In thread: T2: count = 3
```

## `thread-safe`

- The above sample code is not thread safe

- Even though Python has a GIL, operations like:

  ``` python
  count += 1
  ```

  are not atomic. They expand into:

  - load count

  - add 1

  - store count

  Two threads can interleave and cause race conditions.

- We can use lock to solve race condition

``` python
lock = threading.Lock()

def try_count(sleep_seconds, name):
    global count
    with lock:
        count += 1

    time.sleep(sleep_seconds)
    print(f"\nIn thread: {name}: count = {count}")
```

## `thread.local` for per-thread state

- If you want per-thread state instead of shared state use `threading.local()`

``` python
import threading
import time

thread_local = threading.local()

def try_count(sleep_seconds, name):
    thread_local.count = getattr(thread_local, "count", 0)
    thread_local.count += 1
    time.sleep(sleep_seconds)
    print(f"\nIn thread: {name}: count = {thread_local.count}")


t1 = threading.Thread(target=try_count, args=(1, "T1"))
t2 = threading.Thread(target=try_count, args=(2, "T2"))
t3 = threading.Thread(target=try_count, args=(1, "T3"))

t1.start()
t2.start()
t3.start()
```

```
In thread: T1: count = 1
In thread: T3: count = 1
In thread: T2: count = 1
```

## What is `os.fork`

``` mermaid
        graph TD
    Parent[Parent Process<br>pid = os.fork] -->|returns child PID| ParentCont[Parent continues<br>with pid != 0]
    Parent -->|fork| Child[Child Process<br>exact copy of parent]
    Child -->|returns 0| ChildCont[Child runs<br>with pid == 0]
    Parent -->|shares at fork time| Mem[Memory / File Descriptors<br>Code / Open Files]
    Child -->|copy-on-write| Mem
    style Parent fill:#9cf,stroke:#333
    style Child fill:#9f9,stroke:#333
```

``` python
import os

pid = os.fork()

if pid == 0:
    print("I am the child process!")
else:
    print(f"I am the parent process. My child has PID {pid}")
```

## Use case: Pipes for Parent–Child Communication

Python’s `os.pipe()` creates two file descriptors:

- `r` — read end

- `w` — write end

After `fork()`, the parent and child each inherit both ends, so you must close the unused ends in each process.

``` mermaid
        graph LR
    subgraph Pipe[os.pipe - r, w]
        W[Write end w] --> R[Read end r]
    end
    subgraph Parent[Parent Process]
        PW[keep write end w] -->|os.write| W
        PR[close read end r]
    end
    subgraph Child[Child Process]
        CR[keep read end r] -->|os.read| R
        CW[close write end w]
    end
    style Parent fill:#9cf,stroke:#333
    style Child fill:#9f9,stroke:#333
    style Pipe fill:#ffc,stroke:#999
```

``` python
import os

r, w = os.pipe()

pid = os.fork()

if pid == 0:
    # --- Child process ---
    os.close(r)
    message = b"Hello from child!"
    os.write(w, message)
    os.close(w)
else:
    # --- Parent process ---
    os.close(w)
    data = os.read(r, 1024)
    print("Parent received:", data.decode())
    os.close(r)
```

## Use case: Bidirectional communication

``` python
import os
import time

pr, pw = os.pipe()  # parent writes, child reads
cr, cw = os.pipe()  # child writes, parent reads

pid = os.fork()

if pid == 0:
    # --- Child ---
    os.close(pw)
    os.close(cr)
    msg = os.read(pr, 1024)
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
```

## Use case: call a function from child process

``` mermaid
        graph TD
    Fork[os.fork] -->|pid == 0| Child[Child Process]
    Fork -->|pid != 0| Parent[Parent Process]
    Child --> CallFunc[call my_func 10 20]
    Child --> Exit[os._exit 0]
    Parent --> PrintPID[print child PID]
    style Child fill:#9f9,stroke:#333
    style Parent fill:#9cf,stroke:#333
```

``` python
import os

def my_func(x, y):
    print(f"Child running my_func with {x=} and {y=}")

pid = os.fork()

if pid == 0:
    my_func(10, 20)
    os._exit(0)
else:
    print(f"Parent: child PID is {pid}")
```

## Free-Threaded CPython

Python 3.13 introduced **experimental** free-threaded builds (PEP 703), and Python 3.14 made them **officially supported** (PEP 779). This is the biggest change to Python’s concurrency model in a decade.

### What Changed

The GIL (Global Interpreter Lock) can now be **disabled entirely** in free-threaded builds, allowing true thread-level parallelism for CPU-bound work:

``` python
# Standard CPython (with GIL): threads take turns on one core
# Free-threaded CPython (no GIL): threads run on separate cores simultaneously

import threading
import time

def cpu_work():
    total = 0
    for i in range(50_000_000):
        total += i
    return total

# With free-threaded Python, these run in TRUE parallel
t1 = threading.Thread(target=cpu_work)
t2 = threading.Thread(target=cpu_work)

start = time.time()
t1.start(); t2.start()
t1.join(); t2.join()
print(f"Time: {time.time() - start:.2f}s")
# GIL build: ~4s (sequential)
# Free-threaded: ~2s (parallel on 2 cores)
```

### How to Use It

``` bash
# Check if your Python is free-threaded
python -c "import sys; print(sys.flags.nogil)"  # True = free-threaded

# Install free-threaded Python (via uv)
uv python install 3.14t  # 't' suffix = free-threaded

# Run with free-threaded mode
python3.14t your_script.py
```

### When to Use Free-Threaded vs Multiprocessing

| Scenario | Recommendation |
|----|----|
| CPU-bound, shared memory needed | **Free-threaded** — threads share memory natively |
| CPU-bound, isolated processes | **Multiprocessing** — still valid, stronger isolation |
| I/O-bound, high concurrency | **asyncio** — still the best for 10K+ concurrent connections |
| Legacy code with thread-unsafe C extensions | **GIL build** — some C extensions aren’t thread-safe yet |

Free-threaded Python is production-ready in 3.14, but not all third-party C extensions support it yet. Check library compatibility before adopting. For I/O-bound web servers (FastAPI + asyncio), the GIL was never a bottleneck — asyncio remains the right choice.

### Performance

Python 3.14 free-threaded builds have only **5-10% single-thread overhead** compared to GIL builds — a major improvement from 3.13’s initial ~40% overhead.

------------------------------------------------------------------------

## Child process will be copied global variables from parent

``` mermaid
        graph TD
    subgraph Parent[Parent Process]
        PG[global_var = 42<br>shared_list = ...]
    end
    Parent -->|os.fork - copy-on-write| Child[Child Process]
    subgraph Child[Child Process]
        CG[global_var = 42<br>shared_list = ...<br>copied from parent]
    end
    PG -->|child reads same value at fork| CG
    Child -->|modifies its own copy| CM[child global_var = 99<br>parent unaffected]
    style Parent fill:#9cf,stroke:#333
    style Child fill:#9f9,stroke:#333
```
