# Foundations Assessment Quiz

This quiz covers all topics from the Foundations tier. Answer each question, then check your answers using the expandable sections.

------------------------------------------------------------------------

## Section 1: Web Concepts

**1. What are the four components of an HTTP request?**

- A\) URL, Body, Cookie, Session

- B\) Method, URL, Headers, Body

- C\) Protocol, Domain, Port, Path

- D\) GET, POST, PUT, DELETE

Answer

**B** — An HTTP request consists of a method (GET, POST, etc.), URL, headers (metadata like Content-Type, Authorization), and an optional body (for POST/PUT).

**2. What is the difference between a `200`` ``OK` and a `201`` ``Created` response?**

Answer

`200`` ``OK` indicates a successful request (typically GET, PUT, PATCH). `201`` ``Created` indicates a new resource was successfully created (typically POST). The distinction helps clients understand what happened without parsing the response body.

**3. Explain the difference between stateful and stateless communication. Why is HTTP stateless, and how do web applications maintain state?**

Answer

**Stateless**: The server does not retain any information about previous requests — each request is independent. HTTP is stateless by design for simplicity and scalability.

**State is maintained through**: cookies (session IDs), JWT tokens in headers, URL parameters, or server-side session stores (Redis). The client includes state information in each request.

------------------------------------------------------------------------

## Section 2: Threading and Processes

**4. What is the GIL (Global Interpreter Lock) in CPython and how does it affect multithreading?**

- A\) It makes Python threads run in true parallel on multiple cores

- B\) It prevents multiple threads from executing Python bytecode simultaneously, limiting CPU-bound parallelism

- C\) It locks the entire filesystem during thread execution

- D\) It only affects multiprocessing, not threading

Answer

**B** — The GIL ensures only one thread executes Python bytecode at a time. This means CPU-bound work does not benefit from threading in CPython. I/O-bound work (network, disk) still benefits because the GIL is released during I/O waits.

**5. When would you use `threading` vs `multiprocessing` vs `asyncio` in Python?**

Answer

| Approach | Best For | Why |
|----|----|----|
| **threading** | I/O-bound tasks with blocking libraries | Threads release the GIL during I/O |
| **multiprocessing** | CPU-bound tasks (data processing, ML) | Each process has its own GIL |
| **asyncio** | High-concurrency I/O (web servers, API clients) | Single-threaded, no GIL contention, lowest overhead |

**6. What is the difference between `fork()` and spawning a new thread?**

Answer

`fork()` creates a new **process** — a complete copy of the parent process with its own memory space and GIL. A **thread** shares the same memory space as the parent. Processes are isolated (safer, no shared state bugs) but have higher overhead. Threads are lightweight but require synchronization for shared data.

------------------------------------------------------------------------

## Section 3: asyncio

**7. What is the difference between `await` and `asyncio.create_task()`?**

- A\) They are identical

- B\) `await` blocks the coroutine until the result is ready; `create_task()` schedules it to run concurrently

- C\) `create_task()` runs in a separate thread

- D\) `await` is for I/O, `create_task()` is for CPU work

Answer

**B** — `await` suspends the current coroutine until the awaited coroutine completes (sequential). `create_task()` schedules the coroutine to run concurrently on the event loop, allowing multiple coroutines to make progress at the same time.

``` python
# Sequential (slow): total = 2 seconds
result1 = await fetch_url("https://api1.com")  # 1 sec
result2 = await fetch_url("https://api2.com")  # 1 sec

# Concurrent (fast): total = 1 second
task1 = asyncio.create_task(fetch_url("https://api1.com"))
task2 = asyncio.create_task(fetch_url("https://api2.com"))
result1, result2 = await task1, await task2
```

**8. What happens if you call a regular blocking function (like `time.sleep(5)`) inside an async function?**

Answer

It blocks the **entire event loop** for 5 seconds, preventing all other coroutines from running. This defeats the purpose of async. Use `await`` ``asyncio.sleep(5)` instead, or run blocking code in an executor: `await`` ``asyncio.to_thread(time.sleep,`` ``5)`.

------------------------------------------------------------------------

## Section 4: File Descriptors and Sockets

**9. What is a file descriptor and why is it important for network programming?**

Answer

A file descriptor is an integer handle that the OS uses to identify open I/O resources (files, sockets, pipes). In network programming, a socket is represented as a file descriptor, allowing the same system calls (`read`, `write`, `close`) to work with both files and network connections. This is the foundation of the Unix “everything is a file” philosophy.

**10. What is non-blocking I/O and how does it relate to event loops?**

- A\) Non-blocking I/O waits for data to arrive before returning

- B\) Non-blocking I/O returns immediately (even with no data), and the event loop uses selectors to check when data is ready

- C\) Non-blocking I/O creates a new thread for each connection

- D\) Non-blocking I/O is the same as multiprocessing

Answer

**B** — Non-blocking I/O system calls return immediately instead of waiting. The event loop registers sockets with a selector (epoll/kqueue) and is notified when data is ready to read/write. This allows a single thread to handle thousands of concurrent connections.

------------------------------------------------------------------------

## Section 5: CPython Internals

**11. How does CPython’s standard library provide both fast C implementations and portable Python fallbacks?**

Answer

CPython uses an “accelerator” pattern: critical standard library modules have two implementations — a pure Python version (always available) and a C extension that is faster. At import time, Python tries to load the C version first and falls back to the pure Python version if the C extension is not available. Examples: `_json`/`json`, `_pickle`/`pickle`, `_collections`/`collections`.

**12. Why does `import`` ``json` run fast in CPython even though JSON parsing is complex?**

Answer

Because `json` in CPython is backed by `_json`, a C extension module. The heavy parsing and serialization work is done in compiled C code, not interpreted Python bytecode. This is why CPython’s standard library is surprisingly fast for many operations — the “Python” code you call is often a thin wrapper around optimized C.
