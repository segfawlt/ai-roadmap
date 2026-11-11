# CPython vs Pure Python

In CPython (the standard Python implementation), you’ll often find **two versions of the same module or object**:

- **A fast implementation in C**

- **A fallback or reference implementation in pure Python**

This happens for several important reasons:

## **1. Performance (C version)**

Many Python modules, especially those used frequently or in performance-critical code, are implemented in C to make them **much faster**.

Examples:

- `collections.deque` (C)

- `json` → uses `json` (Python) + `json._json` / `ujson`-like C accelerators

- `threading.local` → C implementation (`_thread._local`) + pure Python fallback (`threading.local`)

The C version gives **significant speed improvements**, especially for low-level operations like attribute access, locking, or memory management.

## **2. Portability (Python version)**

CPython can run on many platforms where the C extension might not be available or cannot be compiled. A **pure-Python implementation is guaranteed to run everywhere**, including:

- rare architectures

- restricted environments (embedded systems)

- platforms where compiling C extensions is difficult

So the Python version ensures **broad compatibility**.

## **3. Maintainability and clarity**

The Python version often acts as a **reference implementation**:

- It’s easier to read and understand

- It clarifies tricky semantics

- It helps test and validate the C version for correctness

Example: `threading.local` in Python acts as a readable reference to guide the C implementation.

## **4. Extensibility and fallback**

Python users can sometimes subclass or modify the Python implementation when they can’t modify the C one. Also, if the C extension fails to load, Python seamlessly falls back to the pure version.

------------------------------------------------------------------------

## Example: `threading.local`

The CPython source includes:

| Component | Role |
|----|----|
| `PyThread_local` (C version) | Fast thread-local storage used by default |
| `threading.local` (Python wrapper) | Pure Python fallback + nicer API |

This design lets CPython:

- use the **fast C implementation** in normal conditions,

- but still provide a **pure Python fallback** that behaves the same.

------------------------------------------------------------------------

## CPython 3.14

### Experimental JIT Compiler

Python 3.14 ships with an experimental **JIT (Just-In-Time) compiler** that can improve performance by up to 20% on some workloads. The JIT compiles frequently-executed bytecode into machine code at runtime.

``` bash
# Enable JIT (opt-in via environment variable)
PYTHON_JIT=1 python my_script.py

# Check if JIT is available
python -c "import sys; print(hasattr(sys, '_jit'))"
```

The JIT is transparent — your code runs exactly the same, just faster for hot paths. It is disabled by default and does not affect correctness.

### Deferred Evaluation of Annotations (PEP 649/749)

Python 3.14 changes how type annotations are evaluated. Previously, annotations were evaluated eagerly at function definition time. Now they are **deferred** — stored as special functions and only evaluated when accessed.

``` python
# This now works without quotes (even though MyClass isn't defined yet)
def create() -> MyClass:
    ...

class MyClass:
    pass

# Annotations are evaluated lazily when you access them
import inspect
inspect.get_annotations(create)  # Evaluates at this point
```

**Impact:** Forward references in type hints work without `from`` ``__future__`` ``import`` ``annotations`. Pydantic and FastAPI have already adapted to this change.

### Free-Threaded Mode (PEP 779)

See [Threads vs Processes](threading-and-processes.md) for the full explanation of Python 3.14’s GIL-free threading support.
