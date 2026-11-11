# socket_server_sample.py

```
import selectors
import socket

sel = selectors.DefaultSelector()

# --- Step 1: Create listening socket ---
port = 5555
server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.bind(("0.0.0.0", port))
server_sock.listen()
server_sock.setblocking(False)

# Register the server socket for "read" events (incoming connections)
sel.register(server_sock, selectors.EVENT_READ)

import code
code.interact(local=locals())


def accept_connection(sock):
    client_sock, addr = sock.accept()  # non-blocking accept
    print(f"Accepted connection from {addr}")
    client_sock.setblocking(False)
    # Register client for read events
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


print(f"Server listening on port {port}...")

# --- Step 2: Main event loop ---
while True:
    events = sel.select()  # waits until some socket is ready

    print(f"All events: {events}")


    # print(f"KKK: {sel._fd_to_key}")

    for key, mask in events:
        sock = key.fileobj

        if sock is server_sock:
            print("Accepted socket")
            accept_connection(sock)
        else:
            handle_client(sock)

```## event_loop_with_stop_signal.py
```python

import threading
import asyncio
import signal
import time

stop_event = threading.Event()  # Event to signal thread stop

def thread_worker():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    async def periodic():
        while not stop_event.is_set():  # keep running until stop_event is set
            print("Tick from worker thread")
            await asyncio.sleep(1)



    # Stop loop when stop_event is set
    def check_stop():
        if stop_event.is_set():
            loop.stop()
        else:
            loop.call_later(0.1, check_stop)  # check again after 0.1s
            # pass


    # Other way to implement `check_stop`


    # async def check_stop_task():
    #     while not stop_event.is_set():
    #         await asyncio.sleep(0.1)

    #     loop.stop()

    # loop.create_task(check_stop_task())



    loop.call_soon(check_stop)
    loop.create_task(periodic())



    loop.run_forever()
    # loop.run_until_complete(periodic())



    loop.close()
    print("Worker thread stopped.")

# Start worker thread
t = threading.Thread(target=thread_worker, name="worker")
t.start()

# Signal handler (Ctrl+C sends SIGINT; terminals do not send Ctrl+X as a signal)
def signal_handler(sig, frame):
    if sig == signal.SIGINT:
        print("\nCtrl+C pressed, stopping...")
    elif sig == signal.SIGTERM:
        print("\nSIGTERM received, stopping...")
    stop_event.set()

signal.signal(signal.SIGINT, signal_handler)
signal.signal(signal.SIGTERM, signal_handler)

# Keep main thread alive without busy-waiting
try:
    # Wait until stop_event is set by a signal
    while not stop_event.wait(0.1):
        pass
except KeyboardInterrupt:
    # Fallback: in case default KeyboardInterrupt is raised
    stop_event.set()

t.join()
print("Main thread exiting.")

```## event_loops_in_threads.py
```python
import threading
import asyncio

def thread_worker():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    async def run():
        print("Hello from worker loop in thread:", threading.current_thread().name)

    loop.run_until_complete(run())
    loop.close()


from datetime import datetime
def thread_run_forever():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    async def _run_forever():
        while True:
            await asyncio.sleep(1)

            print(f"Tick: {datetime.now()}")

    loop.create_task(_run_forever())
    loop.run_forever()



# Main thread
main_loop = asyncio.new_event_loop()
asyncio.set_event_loop(main_loop)

# Spawn worker thread
t = threading.Thread(target=thread_worker)
t.start()

async def main():
    print("Hello from main loop in thread:", threading.current_thread().name)

main_loop.run_until_complete(main())
main_loop.close()
t.join()


# Run thread forever ?
t_run_forever = threading.Thread(target=thread_run_forever)
t_run_forever.start()
t_run_forever.join()
```## test.py
```python
import os
import stat
import socket
import fcntl

def describe_fd(fd):
    info = {"fd": fd}

    try:
        st = os.fstat(fd)
        mode = st.st_mode
        info["mode"] = mode
    except OSError:
        return None  # closed fd

    # ---- Determine type --------
    if stat.S_ISREG(mode):
        info["type"] = "regular file"
    elif stat.S_ISDIR(mode):
        info["type"] = "directory"
    elif stat.S_ISCHR(mode):
        info["type"] = "char device"
    elif stat.S_ISBLK(mode):
        info["type"] = "block device"
    elif stat.S_ISFIFO(mode):
        info["type"] = "pipe"
    elif stat.S_ISSOCK(mode):
        info["type"] = "socket"
    elif stat.S_ISLNK(mode):
        info["type"] = "symlink"
    else:
        info["type"] = "unknown"

    # ---- Target path (Linux only) ----
    path = None
    proc_path = f"/proc/self/fd/{fd}"
    if os.path.exists(proc_path):
        try:
            path = os.readlink(proc_path)
        except Exception:
            pass
    info["path"] = path

    # ---- Blocking / nonblocking flags ----
    try:
        flags = fcntl.fcntl(fd, fcntl.F_GETFL)
        info["flags_raw"] = flags
        info["read_only"] = bool(flags & os.O_RDONLY)
        info["write_only"] = bool(flags & os.O_WRONLY)
        info["nonblocking"] = bool(flags & os.O_NONBLOCK)
    except Exception:
        pass

    # ---- Additional socket details ----
    if info["type"] == "socket":
        try:
            sock = socket.socket(fileno=fd)
            info["sock_family"] = sock.family
            info["sock_type"] = sock.type
            info["sock_protocol"] = sock.proto

            try:
                info["local_addr"] = sock.getsockname()
            except Exception:
                pass

            try:
                info["remote_addr"] = sock.getpeername()
            except Exception:
                pass

        except Exception:
            pass

    return info


def list_all_fds(max_fd=4096):
    results = []
    for fd in range(max_fd):
        try:
            os.fstat(fd)
        except OSError:
            continue  # skip closed fd

        details = describe_fd(fd)
        if details:
            results.append(details)
    return results


# ---- Run it ----
for entry in list_all_fds():
    print(entry)

```## event_loop.py
```python
import asyncio


loop = asyncio.get_event_loop()


def normal_fn():
    print(f"Here is normal function")


async def async_fn():
    print(f"Here is async function")



# loop.run_until_complete(normal_fn) # This will be throw exception
loop.run_until_complete(async_fn()) # This will be throw exception

```## selector_sample.py
```python
import selectors
import socket

sel = selectors.DefaultSelector()

sock = socket.socket()
sock.connect(('example.com', 80))
sock.setblocking(False)

sel.register(sock, selectors.EVENT_READ | selectors.EVENT_WRITE)

events = sel.select(timeout=1)

import code
code.interact(local=locals())

for key, mask in events:
    if mask & selectors.EVENT_READ:
        print("Socket ready for reading")
    if mask & selectors.EVENT_WRITE:
        print("Socket ready for writing")

```## socket_sample.py
```python
import socket

# Create a TCP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connect to example.com on port 80
sock.connect(("google.com", 80))

# Send an HTTP request
sock.send(b"GET / HTTP/1.1\r\nHost: google.com\r\n\r\n")

# Receive some data
data = sock.recv(1024)
print(data)


sock.close()
```
