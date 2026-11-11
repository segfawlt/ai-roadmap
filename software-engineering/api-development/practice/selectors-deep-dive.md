# 1. `selectors.py` and its design

The `selectors` module provides a **high-level, cross-platform I/O multiplexing interface**. It wraps lower-level OS-specific mechanisms:

- `select()`

- `poll()`

- `epoll()` (Linux)

- `kqueue()` (macOS, BSD)

The goal is **cross-platform consistency**, so most users never need to touch the low-level APIs.

The `KqueueSelector` class, for example, wraps the **`kqueue` system call** on BSD/macOS. That system call is exposed in Python through the `select` module.

## 2. What `_selector` is

In your snippet:

``` python
kev_list = self._selector.control(None, max_ev, timeout)
```

- `_selector` is an instance of **`select.kqueue()`**.

- `control()` is the method provided by Python’s `select.kqueue` object, which wraps the OS-level `kevent` system call.

So `_selector.control` is **documented in CPython’s `select` module**:

``` python
import select
kq = select.kqueue()
events = kq.control(None, 1, 0)
```

- `kev_list` is a list of `kevent` objects.

- `control()` can take a list of changes (here `None`), the maximum number of events to return (`max_ev`), and a timeout.

------------------------------------------------------------------------

## Deep Dive

``` mermaid
        graph TD
    kevent["kevent object<br>(READ/ADD event)"] -- monitor fd --> fd["fd<br>(file descriptor)"]
    kevent -- register --> kqueue["KqueueSelector object<br>(kqueue)"]
    kqueue -- "self._selector.control([kev], 0, 0)" --> OS["OS kernel<br>(tracks ready fds)"]
    OS -- "returns ready events" --> kqueue
```

Exactly! You’re describing the typical **event-driven I/O loop** for something like a chat server using `kqueue`. Let me clarify step by step and add some practical details.

------------------------------------------------------------------------

## 1. Objects involved

Suppose you have a socket `sock`:

1.  **Socket (`fd`)**

    - This is your actual communication endpoint (TCP/UDP socket).

    - Provides the file descriptor integer `sock.fileno()`.

2.  **`kevent` object(s)**

    - Created for each type of event you care about: `READ`, `WRITE`, etc.

    - Tells the kernel: “Monitor this fd for these events.”

    - Example:

``` python
kev_read = select.kevent(sock.fileno(), select.KQ_FILTER_READ, select.KQ_EV_ADD)
kev_write = select.kevent(sock.fileno(), select.KQ_FILTER_WRITE, select.KQ_EV_ADD)
```

1.  **`kqueue` object**

    - Manages all registered `kevent`s.

    - Tracks which file descriptors are ready.

    - Example:

``` python
kq = select.kqueue()
kq.control([kev_read, kev_write], 0, 0)  # Register events
```

Once the socket and events are registered, you need a **loop to listen for events**:

``` python
while True:
    # Wait for events, max 10 at a time, timeout 1 second
    events = kq.control(None, 10, 1.0)

    for kev in events:
        fd = kev.ident
        if kev.filter == select.KQ_FILTER_READ:
            # fd is readable → read data
            data = os.read(fd, 4096)  # or sock.recv()
            handle_read(fd, data)
        elif kev.filter == select.KQ_FILTER_WRITE:
            # fd is writable → send pending data
            handle_write(fd)
```

------------------------------------------------------------------------

## Example: Simple Chat-Server

``` mermaid
        graph TD
    fd0["fd0<br>(server_socket: host, port)"]
    fd1["fd1<br>(client_sock 1)"]
    fd2["fd2<br>(client_sock 2)"]
    Client1["Client 1"] --> fd1
    Client2["Client 2"] --> fd2
    fd0 & fd1 & fd2 -- "monitor events" --> kq["kqueue<br>.control(events coming)"]
```

- Server: copy code into `simple-chat-server.py`

  ``` sh
  python simple-chat-server.py
  ```

- Client 1: Open terminal (in MacOS)

``` sh
nc 127.0.0.1 7777
```

- Client 2: Open terminal (in MacOS)

``` sh
nc 127.0.0.1 7777
```

## Source code

``` python
import socket
import select
import errno

HOST = '127.0.0.1'
PORT = 7777

# --- Step 1: Create server socket ---
server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_sock.bind((HOST, PORT))
server_sock.listen()
server_sock.setblocking(False)  # important for event-driven I/O

print(f"Chat server running on {HOST}:{PORT}")

# --- Step 2: Create kqueue ---
kq = select.kqueue()

# Register server socket for READ events (new connections)
kev = select.kevent(server_sock.fileno(),
                    filter=select.KQ_FILTER_READ,
                    flags=select.KQ_EV_ADD)
kq.control([kev], 0, 0)

# Map file descriptor → socket object
fd_to_socket = {server_sock.fileno(): server_sock}

# --- Step 3: Event loop ---
while True:
    # Wait for events, max 10 at a time, timeout 1 second
    events = kq.control(None, 10, 1.0)

    for kev in events:
        fd = kev.ident
        sock = fd_to_socket[fd]

        if sock is server_sock:
            # --- New client connection ---
            client_sock, addr = server_sock.accept()
            client_sock.setblocking(False)
            fd_to_socket[client_sock.fileno()] = client_sock

            # Register client for READ events
            kev_client = select.kevent(client_sock.fileno(),

                                       filter=select.KQ_FILTER_READ,
                                       flags=select.KQ_EV_ADD)
            kq.control([kev_client], 0, 0)

            print(f"Client connected from {addr}")

        else:
            # --- Client sent data ---
            try:
                data = sock.recv(4096)
                if data:
                    # Broadcast to all other clients
                    msg = data.decode().strip()
                    print(f"Received from {fd}: {msg}")
                    for other_fd, other_sock in fd_to_socket.items():
                        if other_sock is not server_sock and other_sock is not sock:
                            try:
                                other_sock.sendall(f"(From: {fd}): {msg}\n".encode('utf-8'))
                            except BlockingIOError:
                                pass
                else:
                    # Client closed connection
                    print(f"Client {fd} disconnected")
                    kq.control([select.kevent(fd,
                                               filter=select.KQ_FILTER_READ,
                                               flags=select.KQ_EV_DELETE)], 0, 0)
                    sock.close()
                    del fd_to_socket[fd]

            except (BlockingIOError, InterruptedError):
                pass
```

Perfect! Let’s write a **simple educational chat server** using `kqueue` in Python. This will illustrate the whole workflow: socket creation, `kevent` registration, and the event loop.

This server will:

- Accept multiple clients.

- Broadcast messages from one client to all others.

- Use **non-blocking sockets** and `kqueue`.

------------------------------------------------------------------------

``` python
import socket
import select
import errno

HOST = '127.0.0.1'
PORT = 12345

# --- Step 1: Create server socket ---
server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_sock.bind((HOST, PORT))
server_sock.listen()
server_sock.setblocking(False)  # important for event-driven I/O

print(f"Chat server running on {HOST}:{PORT}")

# --- Step 2: Create kqueue ---
kq = select.kqueue()

# Register server socket for READ events (new connections)
kev = select.kevent(server_sock.fileno(),
                    filter=select.KQ_FILTER_READ,
                    flags=select.KQ_EV_ADD)
kq.control([kev], 0, 0)

# Map file descriptor → socket object
fd_to_socket = {server_sock.fileno(): server_sock}

# --- Step 3: Event loop ---
while True:
    # Wait for events, max 10 at a time, timeout 1 second
    events = kq.control(None, 10, 1.0)

    for kev in events:
        fd = kev.ident
        sock = fd_to_socket[fd]

        if sock is server_sock:
            # --- New client connection ---
            client_sock, addr = server_sock.accept()
            client_sock.setblocking(False)
            fd_to_socket[client_sock.fileno()] = client_sock

            # Register client for READ events
            kev_client = select.kevent(client_sock.fileno(),
                                       filter=select.KQ_FILTER_READ,
                                       flags=select.KQ_EV_ADD)
            kq.control([kev_client], 0, 0)

            print(f"Client connected from {addr}")

        else:
            # --- Client sent data ---
            try:
                data = sock.recv(4096)
                if data:
                    # Broadcast to all other clients
                    msg = data.decode().strip()
                    print(f"Received from {fd}: {msg}")
                    for other_fd, other_sock in fd_to_socket.items():
                        if other_sock is not server_sock and other_sock is not sock:
                            try:
                                other_sock.sendall(data)
                            except BlockingIOError:
                                pass
                else:
                    # Client closed connection
                    print(f"Client {fd} disconnected")
                    kq.control([select.kevent(fd,
                                               filter=select.KQ_FILTER_READ,
                                               flags=select.KQ_EV_DELETE)], 0, 0)
                    sock.close()
                    del fd_to_socket[fd]

            except (BlockingIOError, InterruptedError):
                pass
```

## How it works

1.  **Server socket**:

    - Non-blocking, registered with `kqueue` for **READ events** (incoming connections).

2.  **New client**:

    - Accept connection, set non-blocking, register client fd with `kqueue`.

3.  **Event loop**:

    - Call `kqueue.control(None,`` ``max_events,`` ``timeout)` to get ready events.

    - For server fd → accept new client.

    - For client fd → read data and broadcast.

4.  **Client disconnect**:

    - Remove fd from `kqueue` and close the socket.

- This is **single-threaded, event-driven**, no blocking anywhere.

- You could easily add **WRITE events** if you want to handle buffered outgoing messages separately.

- Perfect for educational purposes to see **kevent → kqueue → event loop** in action.

------------------------------------------------------------------------

## Cleaner version

- using `selectors`

``` python
import selectors
import socket

HOST = "127.0.0.1"
PORT = 12345

# --- Step 1: Create selector ---
sel = selectors.DefaultSelector()

# --- Step 2: Accept new client connections ---
def accept(sock):
    client_sock, addr = sock.accept()
    print("Connected from:", addr)
    client_sock.setblocking(False)
    sel.register(client_sock, selectors.EVENT_READ, read_client)

# --- Step 3: Read client data ---
def read_client(client_sock):
    try:
        data = client_sock.recv(4096)
    except BlockingIOError:
        return

    if data:
        msg = data.decode().strip()
        print(f"[{client_sock.fileno()}] {msg}")

        # Broadcast to all clients
        for key, mask in sel.get_map().items():
            s = key.fileobj
            if s not in (server_sock, client_sock):
                try:
                    s.sendall(data)
                except BlockingIOError:
                    pass
    else:
        # Client disconnected
        print(f"Client {client_sock.fileno()} disconnected")
        sel.unregister(client_sock)
        client_sock.close()

# --- Step 4: Create server socket ---
server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_sock.bind((HOST, PORT))
server_sock.listen()
server_sock.setblocking(False)

sel.register(server_sock, selectors.EVENT_READ, accept)

print(f"Chat server running on {HOST}:{PORT}")

# --- Step 5: Event loop ---
while True:
    events = sel.select(timeout=1)  # blocks until events ready
    for key, mask in events:
        callback = key.data     # accept or read_client
        sock = key.fileobj
        callback(sock)
```
