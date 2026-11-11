# Event Loop

``` python
import asyncio


loop = asyncio.get_event_loop()


def normal_fn():
    print(f"Here is normal function")


async def async_fn():
    print(f"Here is async function")


# loop.run_until_complete(normal_fn) # This will throw an exception
loop.run_until_complete(async_fn())

print(loop)
```

## Best Practice

``` python
asyncio.run(my_async_fn())
```

## Diagram

``` mermaid
        flowchart TD
    P["Python Program"] --> T["Main OS Thread"]
    T --> A["Loop A<br>(new_event_loop)<br>Idle"]
    T --> B["Loop B<br>(new_event_loop)<br>Idle"]
    T --> C["Loop C<br>(new_event_loop)<br>Idle"]
    A --> R["Running Event Loop (e.g., A)<br>Executes async tasks<br>Handles timers / sockets<br>Manages await points"]
    R -. "Only ONE loop can be<br>RUNNING in this thread" .-> A
```

## Threads and Event loop

``` mermaid
        flowchart TD
    MT["Main Thread"] --> LA["Event Loop A<br>(running or idle)"]
    WT["Worker Thread 1"] --> LB["Event Loop B<br>(running or idle)"]
```

### Important asyncio rules

**Rule 1 — One running event loop per thread** Each thread may have one running event loop.

**Rule 2 — Event loop is NOT shared between threads** If you try to use a loop created in Thread A while in Thread B → RuntimeError.

**Rule 3 — Threads communicate via thread-safe asyncio functions**

``` python
loop.call_soon_threadsafe(callback, arg)
```

### Examples

``` python
import threading
import asyncio

def thread_worker():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    async def run():
        print("Hello from worker loop in thread:", threading.current_thread().name)

    loop.run_until_complete(run())
    loop.close()


# Main thread
main_loop = asyncio.new_event_loop()
asyncio.set_event_loop(main_loop)

t = threading.Thread(target=thread_worker)
t.start()

async def main():
    print("Hello from main loop in thread:", threading.current_thread().name)

main_loop.run_until_complete(main())
main_loop.close()
t.join()
```

## Implement signal stop

``` mermaid
        graph TD
    Main[Main Thread] -->|sets stop_event| SE[threading.Event<br>stop_event]
    SH[signal_handler<br>SIGINT / SIGTERM] -->|stop_event.set| SE
    subgraph WorkerThread[Worker Thread]
        EL[Event Loop] --> CS[call_soon check_stop]
        CS -->|every 0.1s| CheckStop{stop_event<br>set?}
        CheckStop -->|No| CL[call_later 0.1s check_stop]
        CL --> CheckStop
        CheckStop -->|Yes| Stop[loop.stop]
        EL --> PT[periodic task<br>print Tick]
    end
    SE -->|is_set| CheckStop
    style SE fill:#f96,stroke:#333
    style WorkerThread fill:#eef,stroke:#669
```

``` python
import threading
import asyncio
import signal
import time

stop_event = threading.Event()

def thread_worker():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    async def periodic():
        while not stop_event.is_set():
            print("Tick from worker thread")
            await asyncio.sleep(1)

    def check_stop():
        if stop_event.is_set():
            loop.stop()
        else:
            loop.call_later(0.1, check_stop)

    loop.call_soon(check_stop)
    loop.create_task(periodic())
    loop.run_forever()
    loop.close()
    print("Worker thread stopped.")

t = threading.Thread(target=thread_worker, name="worker")
t.start()

def signal_handler(sig, frame):
    if sig == signal.SIGINT:
        print("\nCtrl+C pressed, stopping...")
    elif sig == signal.SIGTERM:
        print("\nSIGTERM received, stopping...")
    stop_event.set()

signal.signal(signal.SIGINT, signal_handler)
signal.signal(signal.SIGTERM, signal_handler)

try:
    while not stop_event.wait(0.1):
        pass
except KeyboardInterrupt:
    stop_event.set()

t.join()
print("Main thread exiting.")
```

- `call_soon` and `call_later` are used to schedule `check_stop` which monitors the `stop_signal` event

- `check_stop` registers itself to keep checking the signal every 0.1 seconds

- An alternative approach uses a task with a loop instead of `call_soon`/`call_later`

``` python
async def check_stop_task():
    while not stop_event.is_set():
        await asyncio.sleep(0.1)
    loop.stop()

loop.create_task(check_stop_task())
```

### Event loop internals

``` mermaid
        graph TD
    Loop[Event Loop Iteration] --> Phase1[Phase 1: Run immediate callbacks<br>loop._ready queue]
    Phase1 --> Phase2[Phase 2: Wait for I/O<br>selector.select timeout]
    Phase2 -->|I/O events ready| Convert[Convert I/O events<br>to immediate callbacks]
    Convert --> Phase3[Phase 3: Move expired timers<br>loop._scheduled into _ready]
    Phase3 --> Loop
    style Loop fill:#9cf,stroke:#333
    style Phase1 fill:#9f9,stroke:#333
    style Phase2 fill:#fc9,stroke:#333
    style Phase3 fill:#f9c,stroke:#333
```

``` python
while True:

    # Phase 1 — RUN immediate callbacks
    while loop._ready:
        handle = loop._ready.pop(0)
        handle._run()

    # Phase 2 — WAIT for I/O until next timer
    timeout = compute_timeout_from_scheduled()
    events = selector.select(timeout)

    # Convert I/O events into immediate callbacks
    for event in events:
        loop._ready.append(event.handle)

    # Phase 3 — MOVE expired timers into ready queue
    now = loop.time()
    while loop._scheduled and loop._scheduled[0].when <= now:
        timer = heapq.heappop(loop._scheduled)
        loop._ready.append(timer)
```

## `event-loop` is thread-based

``` python
import asyncio
import threading

def worker():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    print("Worker thread loop:", id(asyncio.get_event_loop()))


threading.Thread(target=worker).start()

main_loop = asyncio.new_event_loop()
asyncio.set_event_loop(main_loop)
print("Main thread loop:", id(asyncio.get_event_loop()))
```

## `running-loop` is the `event-loop` that is running

``` python
import asyncio
import threading

import nest_asyncio
nest_asyncio.apply()


async def report():
    print(f"Hey man, I'm running inside {threading.current_thread()}")
    print(f"Running loop: {id(asyncio.get_running_loop())}. Current loop: {id(asyncio.get_event_loop())}")


def loop_run_to_complete(loop):
    loop.create_task(report())
    loop.run_until_complete(asyncio.sleep(1.0))

def worker():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    loop_run_to_complete(loop)


threading.Thread(target=worker).start()

main_loop = asyncio.new_event_loop()
asyncio.set_event_loop(main_loop)
loop_run_to_complete(main_loop)
```

```
Hey man, I'm running inside <Thread(Thread-10 (worker), started 6305296384)>
Running loop: 4366786128. Current loop: 4366786128
Hey man, I'm running inside <_MainThread(MainThread, started 8697635008)>
Running loop: 4366783392. Current loop: 4366783392
```

## Socket Server Sample

**Key concepts:**

- Create a server that exposes a `server_socket` listening on a specified port

- The `server_socket` is registered in a `selector` object for I/O event bookkeeping

- A main loop monitors all incoming events and processes them:

  - **Accept connection**: When a client first connects, register the client socket

  - **Handle connection**: Read and write data via the registered client socket

``` mermaid
        graph TD
    Start[Create server_socket<br>bind + listen port 5555] --> Register[sel.register server_sock<br>EVENT_READ]
    Register --> MainLoop[Main Loop<br>sel.select]
    MainLoop -->|event on server_sock| Accept[accept_connection<br>accept new client]
    Accept -->|register client_sock| Register
    MainLoop -->|event on client_sock| Handle[handle_client<br>recv and sendall]
    Handle -->|data received| Echo[send Message received]
    Handle -->|no data / disconnect| Cleanup[sel.unregister<br>sock.close]
    Cleanup --> MainLoop
    Echo --> MainLoop
    style MainLoop fill:#9cf,stroke:#333
    style Accept fill:#9f9,stroke:#333
    style Handle fill:#fc9,stroke:#333
```

**Testing:**

- Run the code in server: `python`` ``this-code.py`

- From client (same machine):

  - MacOS: `nc`` ``127.0.0.1`` ``5555`

  - Windows: `telnet`` ``127.0.0.1`` ``5555`

``` python
import selectors
import socket

sel = selectors.DefaultSelector()

# --- Step 1: Create listening socket ---
server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.bind(("0.0.0.0", 5555))
server_sock.listen()
server_sock.setblocking(False)

# Register the server socket for "read" events (incoming connections)
sel.register(server_sock, selectors.EVENT_READ)


def accept_connection(sock):
    client_sock, addr = sock.accept()
    print(f"Accepted connection from {addr}")
    client_sock.setblocking(False)
    sel.register(client_sock, selectors.EVENT_READ)


def handle_client(sock):
    data = sock.recv(1024)
    if data:
        print("Received:", data.decode())
        sock.sendall(b"Message received\n")
    else:
        print("Client disconnected")
        sel.unregister(sock)
        sock.close()


print("Server listening on port 5555...")

# --- Step 2: Main event loop ---
while True:
    events = sel.select()
    for key, mask in events:
        sock = key.fileobj

        if sock is server_sock:
            accept_connection(sock)
        else:
            handle_client(sock)
```
