# Concurrency Code Exploration

These code snippets demonstrate key concurrency concepts from the Foundations tier. Run them in a Jupyter notebook or Python REPL to observe the behavior firsthand.

## Environment Check

Verify your Python environment supports process forking:

``` python
import os

hasattr(os, "register_at_fork")
```

```
True
```

``` python
import sys

sys.platform
```

```
'darwin'
```

## Multiprocessing: True Parallelism

Two processes run `cpu_task` in parallel — observe that total time is roughly the same as a single process (true parallelism bypasses the GIL):

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
Process finished in 1.1804568767547607
Process finished in 1.180586814880371
Total time: 1.3442530632019043
```

``` python
p1, p2
```

```
(<Process name='Process-1' pid=41235 parent=40199 stopped exitcode=0>,
 <Process name='Process-2' pid=41236 parent=40199 stopped exitcode=0>)
```

``` python
p1._parent_name
```

```
'MainProcess'
```

``` python
import os

os.getpid()
```

```
40199
```

## Fork: Creating a Child Process

`os.fork()` duplicates the current process. The child gets `pid=0`, the parent gets the child’s PID. Note the deprecation warning when forking in a multi-threaded context (like Jupyter):

``` python
import os

pid = os.fork()

if pid == 0:
    print("Child process")
else:
    print("Parent process, child PID:", pid)
```

```
Parent process, child PID: 48145


/var/folders/62/3bbgw7552ydbl_jtggnlgln40000gn/T/ipykernel_40199/1265583681.py:3: DeprecationWarning: This process (pid=40199) is multi-threaded, use of fork() may lead to deadlocks in the child.
  pid = os.fork()


Child process


Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages/ipykernel_launcher.py", line 18, in <module>
    app.launch_new_instance()
  File "/Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages/traitlets/config/application.py", line 1075, in launch_instance
    app.start()
  File "/Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages/ipykernel/kernelapp.py", line 758, in start
    self.io_loop.start()
  File "/Users/hotronghai/Documents/Hai-Source/lectures-fs/.venv/lib/python3.12/site-packages/tornado/platform/asyncio.py", line 211, in start
    self.asyncio_loop.run_forever()
  File "/opt/homebrew/Cellar/[email protected]/3.12.2/Frameworks/Python.framework/Versions/3.12/lib/python3.12/asyncio/base_events.py", line 639, in run_forever
    self._run_once()
  File "/opt/homebrew/Cellar/[email protected]/3.12.2/Frameworks/Python.framework/Versions/3.12/lib/python3.12/asyncio/base_events.py", line 1947, in _run_once
    event_list = self._selector.select(timeout)
                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/opt/homebrew/Cellar/[email protected]/3.12.2/Frameworks/Python.framework/Versions/3.12/lib/python3.12/selectors.py", line 566, in select
    kev_list = self._selector.control(None, max_ev, timeout)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ValueError: I/O operation on closed kqueue object
```

## Thread-Local Storage

Each thread gets its own isolated copy of `data.value` — observe different thread IDs:

``` python
import threading

data = threading.local()

def worker():
    data.value = threading.get_ident()
    print(data.value)

threading.Thread(target=worker).start()
threading.Thread(target=worker).start()
```

```
6295334912
6312161280
```
