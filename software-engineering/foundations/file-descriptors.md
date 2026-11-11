# File Descriptors

**In the context of `selectors`, `fileobj` is used *only* for monitoring I/O readiness events.**

Selectors are built around OS-level **file descriptors**, and file descriptors represent resources that support **non-blocking I/O**, such as:

- sockets

- pipes

- files

- character devices

- eventfd / timerfd (on Linux)

- custom objects exposing a `.fileno()` that maps to one of the above

Selectors do **not** care about anything else — only whether a file descriptor is ready for **reading** or **writing**.

## What selectors *do*

A selector monitors file descriptors for events:

- `EVENT_READ` → “ready to read without blocking”

- `EVENT_WRITE` → “ready to write without blocking”

## What selectors *do NOT* do

Selectors do **not**:

- read or write data

- watch arbitrary objects

- handle application logic

- manage timeouts or threading by themselves

They exist purely to efficiently check **I/O readiness** in event loops.

## File Descriptors in macOS and Unix

In macOS (and most Unix-like operating systems), **file descriptors are managed by processes**, not threads.

``` mermaid
        graph TD
    OS[Operating System<br>Kernel] --> FDT1[Process A<br>FD Table]
    OS --> FDT2[Process B<br>FD Table]
    FDT1 --> FD0A[FD 0 - stdin]
    FDT1 --> FD1A[FD 1 - stdout]
    FDT1 --> FD2A[FD 2 - stderr]
    FDT1 --> FD3A[FD 3 - socket / file / pipe]
    subgraph Threads[Threads within Process A]
        T1[Thread 1] -->|shared FD table| FDT1
        T2[Thread 2] -->|shared FD table| FDT1
        T3[Thread 3] -->|shared FD table| FDT1
    end
    style FDT1 fill:#9cf,stroke:#333
    style FDT2 fill:#9f9,stroke:#333
    style Threads fill:#eef,stroke:#669
```

Here’s a breakdown of how it works:

1.  **File Descriptors and Processes:**

    - When a process opens a file or socket, the operating system assigns a **file descriptor** (an integer value) to it. This file descriptor is used to perform operations like reading or writing to the file, or communicating over a network.

    - Each process has its own file descriptor table, which keeps track of the file descriptors and their associated resources (like open files, sockets, etc.).

2.  **Threads and File Descriptors:**

    - Threads within a process share the same file descriptor table, meaning that any thread within a process can use the file descriptors opened by other threads in the same process. This allows multiple threads to access the same files or sockets concurrently.

    - However, file descriptors themselves are not tied to specific threads. Instead, they belong to the process and are shared among all threads of that process.

3.  **Concurrency Considerations:**

    - While threads can access the same file descriptors, you might still need to use synchronization mechanisms (like mutexes) to avoid race conditions if multiple threads are interacting with the same file descriptor simultaneously.

In summary, **file descriptors are managed at the process level**, but threads within a process can share and use them.

## Example 1: List all file descriptors in current process

``` python
import os

os.listdir("/dev/fd")

"""
The output is number that OS uses to identify resources to manage

{
    "0": "Standard Input (stdin)",
    "1": "Standard Output (stdout)",
    "2": "Standard Error (stderr)",
    "3": "File descriptor 3, could be a file, socket, or pipe",
    "4": "File descriptor 4, could be a file, socket, or pipe",
    # Additional file descriptors could follow depending on the environment
}
"""
```

## Example 2: Use `selectors` module to monitor I/O events on sockets

``` mermaid
        graph TD
    Sock[socket.socket<br>AF_INET SOCK_STREAM] -->|bind localhost:12345<br>listen| BoundSock[Bound Socket<br>FD assigned by OS e.g. FD 3]
    BoundSock -->|selector.register<br>EVENT_READ| Sel[DefaultSelector]
    Sel -->|selector.select timeout=1| Wait[Wait for I/O events]
    Wait -->|connection arrives| Event[Event on FD 3]
    Event -->|key.fileobj == sock| Accept[sock.accept<br>get connection + address]
    style Sel fill:#9cf,stroke:#333
    style BoundSock fill:#9f9,stroke:#333
    style Wait fill:#fc9,stroke:#333
```

``` python
import selectors
import socket

# Create a default selector (selects for read/write events)
selector = selectors.DefaultSelector()

# Create a socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Bind and listen
sock.bind(('localhost', 12345))
sock.listen()

# Register the socket to listen for incoming connections
selector.register(sock, selectors.EVENT_READ)

# Blocking call that waits for events
events = selector.select(timeout=1)

for key, events in events:
    if key.fileobj == sock:
        connection, address = sock.accept()
        print(f"Accepted connection from {address}")
```

- **Creating a socket**: When you call `socket.socket()`, the OS allocates a file descriptor for the socket starting from the lowest available FD (usually 3, since 0, 1, and 2 are reserved for stdin, stdout, and stderr).

- **Binding and listening**: `sock.bind()` associates the socket with an IP address and port. The OS manages this via the file descriptor.

- **FD and the `selectors` module**: The socket FD is registered in the kernel’s FD table and can be monitored for I/O events (ready to read or write).

## FD Example with psutil

``` python
import psutil, os

p = psutil.Process(os.getpid())

# list of all open files (regular files)
print("Files:")
for f in p.open_files():
    print(f.fd, f.path)

# list of all socket connections (TCP/UDP)
print("\nSockets:")
for c in p.net_connections():
    print(c.fd, c.family, c.type, c.laddr, c.raddr, c.status)

# total number of FDs
print("\nTotal FD count:", p.num_fds())
```

```
Files:
65 /Users/hotronghai/.ipython/profile_default/history.sqlite
66 /Users/hotronghai/.ipython/profile_default/history.sqlite

Sockets:
14 2 1 addr(ip='127.0.0.1', port=9002) () LISTEN
17 2 1 addr(ip='127.0.0.1', port=9003) () LISTEN
...

Total FD count: 89
```
