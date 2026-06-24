# Flask + Gunicorn gthread on Python 3.14t (Free-Threaded / No-GIL): Complete Guide

---

## Part 1: Tutorial — gthread under Python 3.14t instead of gevent

### Why gthread instead of gevent now?

The traditional reason to reach for gevent was that `gthread` workers were "at the mercy of the GIL" — threads spawned by the same worker share the same memory space and were historically constrained by the GIL. So developers monkeypatched gevent to get cooperative concurrency and avoid that bottleneck.

That trade-off is now gone. Python 3.14's free-threaded build removes the GIL, allowing multiple threads to execute Python bytecode simultaneously — no multiprocessing, no pickle, no hacks. `gthread` on `python3.14t` means your threads are genuinely parallel, not just concurrent. And unlike gevent, there's no monkeypatching required — standard library I/O, `threading`, `queue`, everything works as-is.

---

### Install the Free-Threaded Interpreter

The free-threaded interpreter is not installed by default. On macOS it requires being selected as a customized install, and on Windows the user needs to add it via the new Python install manager.

**Via pyenv (Linux/macOS — recommended for dev):**

```bash
# Install the free-threaded variant (note the 't' suffix)
pyenv install 3.14t

# Set it for your project
pyenv local 3.14t

# Verify — output should say "free-threading build"
python3.14t --version
```

**Via uv (fastest):**

```bash
uv python install 3.14t
uv python pin 3.14t
```

**Programmatic check inside your app:**

```python
import sys

if not sys._is_gil_enabled():
    print("✓ Running on free-threaded Python — GIL is disabled")
else:
    print("⚠ GIL is still active — check your interpreter")
```

> If a third-party C extension hasn't been recompiled for free-threading, it may silently re-enable the GIL for the entire process. The check above lets you catch that at startup.

---

### The gthread Worker Explained

With `gthread`, the Python application is loaded once per worker process, and each of the threads spawned by the same worker shares the same memory space. On `python3.14t`, those threads now run on separate CPU cores simultaneously.

If you set `--threads` to more than 1 without specifying a worker class, Gunicorn automatically switches from `sync` to `gthread`. You can also set it explicitly with `--worker-class gthread`.

The formula for total concurrency is:

```
max simultaneous requests = workers × threads
```

The suggested maximum for the total is still `(2 × CPU_cores) + 1`. So on a quad-core machine, you might use 3 workers × 3 threads = 9 concurrent requests. On free-threaded Python, unlike before, those 9 threads can genuinely run in parallel across cores.

---

### Flask App with Thread Safety

With real parallelism, race conditions that the GIL previously masked will now surface. Flask itself is fine — each request gets its own independent request context and there is no shared state between threads; Flask automatically pushes and pops these contexts as requests are handled.

What you **do** need to guard is any module-level mutable state.

```python
# app.py
from flask import Flask, jsonify, g
import threading
import sqlite3

app = Flask(__name__)

# --- SAFE: per-request context, Flask manages this ---
@app.teardown_appcontext
def close_db(error):
    db = g.pop("db", None)
    if db is not None:
        db.close()

def get_db():
    if "db" not in g:
        g.db = sqlite3.connect("myapp.db", check_same_thread=False)
    return g.db

# --- SAFE: immutable module-level config ---
CONFIG = {"max_items": 100, "version": "1.0"}

# --- UNSAFE without a lock: mutable shared counter ---
_request_count = 0
_count_lock = threading.Lock()

@app.route("/count")
def count():
    global _request_count
    with _count_lock:
        _request_count += 1
        current = _request_count
    return jsonify({"requests": current})

@app.route("/items")
def items():
    db = get_db()
    cursor = db.execute("SELECT * FROM items LIMIT ?", (CONFIG["max_items"],))
    return jsonify({"items": [dict(row) for row in cursor.fetchall()]})
```

**Key rules under free-threaded Python:**

- **Per-request state** → use Flask's `g` object or local variables. Always safe.
- **Immutable globals** → fine as-is (dicts/lists you never mutate after startup).
- **Mutable shared state** → protect with `threading.Lock()` or `threading.RLock()`.
- **Queues** → `queue.Queue` is already thread-safe; use it freely.

---

### Gunicorn Configuration

**`gunicorn.conf.py`** (preferred over inline flags for production):

```python
import multiprocessing
import sys

# Verify we're on the free-threaded build before anything starts
if sys._is_gil_enabled():
    raise RuntimeError(
        "GIL is enabled! Run with python3.14t, not python3.14.\n"
        "Also check for C extensions that may have re-enabled the GIL."
    )

# Worker formula: fewer workers, more threads per worker
# on free-threaded Python, threads are genuinely parallel
cores = multiprocessing.cpu_count()
workers = max(2, cores // 2)          # e.g. 4 cores → 2 workers
threads = cores                        # e.g. 4 cores → 4 threads/worker
# Total concurrency = workers × threads (e.g. 2 × 4 = 8)

worker_class = "gthread"
bind = "0.0.0.0:8000"
timeout = 60
graceful_timeout = 30
keepalive = 5

# Logging
accesslog = "-"
errorlog = "-"
loglevel = "info"

# Prevent memory bloat — restart workers after N requests
max_requests = 1000
max_requests_jitter = 100             # randomise to avoid thundering herd
```

**Launch command:**

```bash
python3.14t -m gunicorn -c gunicorn.conf.py app:app
```

**CLI equivalent (for quick tests):**

```bash
python3.14t -m gunicorn \
  --worker-class gthread \
  --workers 2 \
  --threads 4 \
  --bind 0.0.0.0:8000 \
  --timeout 60 \
  app:app
```

---

### Dependency Compatibility Check

C extensions must be recompiled for free-threading, or they might quietly re-enable the GIL. Run this audit script before deploying:

```python
# check_gil.py
import sys
import importlib

DEPS_TO_CHECK = [
    "flask", "werkzeug", "sqlalchemy",
    "redis", "pydantic", "cryptography",
    # add your own
]

print(f"GIL at start: {sys._is_gil_enabled()}")

for name in DEPS_TO_CHECK:
    before = sys._is_gil_enabled()
    try:
        importlib.import_module(name)
        after = sys._is_gil_enabled()
        status = "⚠ RE-ENABLED GIL" if (not before and after) else "✓ OK"
        print(f"  {name}: {status}")
    except ImportError:
        print(f"  {name}: not installed")

print(f"GIL at end: {sys._is_gil_enabled()}")
```

Run it with:

```bash
python3.14t check_gil.py
```

---

### Workers vs Threads Trade-Off on Free-Threaded Python

| Concern | Old GIL-era advice | With python3.14t |
|---|---|---|
| CPU parallelism | Use more **workers** (processes) | Use more **threads** — they're truly parallel now |
| Memory footprint | Threads save memory vs forks | Same benefit, but threads also do CPU work |
| Shared state | Processes = no sharing needed | Threads share memory — add locks where needed |
| Startup time | Workers are slow to fork | Threads start much faster |
| C extension safety | GIL protected you silently | Audit extensions explicitly |

---

### Quick Reference

```bash
# Install
pyenv install 3.14t && pyenv local 3.14t
pip install flask gunicorn --break-system-packages

# Verify GIL is off
python3.14t -c "import sys; print('GIL off:', not sys._is_gil_enabled())"

# Run
python3.14t -m gunicorn --worker-class gthread --workers 2 --threads 4 app:app
```

---

## Part 2: What gevent Monkeypatching Has Broken — and What Free-Threaded gthread Can Break Instead

These are genuinely different failure modes. Understanding both helps you know whether a problem you hit is old baggage or a new hazard.

---

### The Gevent Monkeypatch Hall of Shame

Gevent's monkeypatch works by replacing standard library objects at runtime — `socket`, `ssl`, `threading`, `os.fork`, `time.sleep`, and others — with cooperative greenlet-aware versions. Because this is a global mutation of Python's module system, it is deeply fragile.

#### 1. The Timing Problem: Patch-Too-Late

This is the most common gevent failure. If `ssl` has already been imported before `monkey.patch_all()` is called, you get a `MonkeyPatchWarning` that patching SSL after import may lead to errors, including `RecursionError`. The reason is subtle: in Python 3.6+, `SSLContext` properties use `super(SSLContext, SSLContext)` to access their setters; when gevent rebinds `SSLContext`, that `super()` chain starts recursing infinitely into itself.

This means any library that imports `ssl` at module load time — which includes `requests`, `urllib3`, APM agents like New Relic or Datadog, and many others — will detonate gevent if it loads before your `patch_all()` call. Datadog's tracing agent, for instance, imports `urllib` before gevent can patch it, producing exactly this `RecursionError` cascade in production.

#### 2. multiprocessing Hangs Permanently

Monkeypatching `thread` and then using `multiprocessing.Queue` or `concurrent.futures.ProcessPoolExecutor` (which internally uses a Queue) will hang the process entirely. This catches many teams off guard because Celery, background job runners, and any code that spawns subprocesses via `multiprocessing` silently deadlocks.

#### 3. psycopg2 Blocks the Event Loop

Standard psycopg2 is not gevent-aware. If you don't apply a separate patch (via the `psycogreen` package), database calls block the entire greenlet event loop, eliminating the concurrency benefit. To use gevent with PostgreSQL, you need psycogreen in addition to psycopg2, and PyMySQL instead of mysqlclient for MySQL, because the C-level MySQL client is not patchable.

#### 4. RLock Patching is Broken on Python 3

Under Python 3, gevent's monkeypatching code fails to find and patch existing `RLock` instances. A lock created before `patch_all()` is called is not converted to a greenlet-cooperative lock, so code that holds an `RLock` across a yield point will silently deadlock.

#### 5. setuptools Dependency Freeze

Setuptools ≥ 60.7.0 introduced a vendored `more_itertools` dependency, which imports `concurrent.futures.threading` at import time — one of the modules gevent patches. This creates a chicken-and-egg problem where gevent cannot patch that module, and the process freezes at startup.

#### 6. Test Suites and Debuggers Silently Break

Patching `threading` globally means debuggers (PyCharm, pdb), test runners (pytest, nose), and profilers that rely on real threading semantics behave incorrectly or hang. The patch has no scope — it's process-wide and permanent. There is no official `unpatch_all()`.

#### Summary of Gevent's Failure Pattern

Every gevent problem flows from the same root cause: **global, irreversible, import-order-sensitive mutation of the standard library**. The patch must happen before any other import, but that's architecturally impossible to guarantee when you don't control all your dependencies.

---

### What Free-Threaded gthread Can Break

The failure modes here are fundamentally different. Instead of import-order chaos, you get **actual concurrency bugs** that the GIL previously masked. These are more predictable in kind, but harder to catch in testing because they are timing-dependent.

#### 1. C Extensions That Assumed the GIL Exists

Many C extensions were built with the assumption that the GIL exists, implicitly protecting them from certain types of race conditions. Running these in a free-threaded environment can lead to crashes and data corruption. The insidious version: the extension silently re-enables the GIL for your whole process, and your `sys._is_gil_enabled()` check at startup is the only way to catch it.

Most libraries need another 6–12 months before most developers can switch without hitting this silent GIL re-enable. Checking compatibility before deploying is essential.

#### 2. Race Conditions That Were Previously Masked

Under the GIL, only one thread executed Python bytecode at a time, so many race conditions never actually fired. Under `python3.14t`, threads genuinely run in parallel. Any code with:

- A read-modify-write on a shared variable (even `+=` on an integer)
- Lazy initialization (`if cache is None: cache = build_cache()`)
- Dict or list mutations shared across requests without a lock

...can now produce real data corruption.

#### 3. SQLAlchemy Session Misuse at Scale

SQLAlchemy has already been considering thread safety in its development and should work with the free-threaded build out of the box, but this doesn't protect you from application-level session misuse. Under high thread concurrency, the `QueuePool` connection pool has a known thread starvation problem: the pool does not fairly distribute connections under high contention, leading to timeouts as the same threads keep acquiring and releasing the same connections while other threads wait past the timeout threshold.

The practical fix is to size `pool_size` and `max_overflow` to match `workers × threads`, and to use per-request sessions via Flask's `g` object rather than any shared session state.

#### 4. asyncpg / asyncio DB Drivers Can Bus Error

There are live reports of `asyncpg` combined with SQLAlchemy's asyncpg dialect triggering a fatal `Bus error` under `python3.14t`, with the crash message "Cannot show all threads while the GIL is disabled." If your Flask app uses any `async` views backed by asyncpg, this is an active hazard to verify before going to production.

#### 5. Logging Under True Parallelism

Python's `logging` module uses internal locks and is documented as thread-safe, but log handlers that write to files or streams without their own buffering can interleave output under genuine parallelism in ways that don't happen with the GIL. Structured logging libraries (loguru, structlog) are generally safer here.

#### 6. Lazy-Initialized Singletons

The classic pattern:

```python
_db_engine = None

def get_engine():
    global _db_engine
    if _db_engine is None:          # two threads can both see None
        _db_engine = create_engine(...)  # two engines get created
    return _db_engine
```

Under the old GIL this was usually safe by accident. Under `python3.14t`, both threads genuinely run the `if` check simultaneously. Use a `threading.Lock` or better yet, initialize singletons at module load time before any threads start.

---

### Side-by-Side Comparison

| Failure type | Gevent monkeypatch | Free-threaded gthread |
|---|---|---|
| SSL/TLS recursion crash | Yes — common in production | No |
| `multiprocessing` deadlock | Yes — documented, hard to fix | No |
| Import order dependency | Yes — brittle, often invisible | No |
| DB driver requires extra patching | Yes (psycogreen, PyMySQL) | No |
| `RLock` / `threading` brokenness | Yes | No |
| APM/tracer agent conflicts | Yes | No |
| Silent GIL re-enable | N/A | Yes — new risk |
| Race conditions from parallel execution | No (cooperative, never truly parallel) | Yes — new risk |
| C extension crashes | No | Yes — new risk |
| DB connection pool starvation | Greenlet exhaustion equivalent | More likely under true parallelism |
| Lazy-init singleton bugs | Rare (GIL masked it) | Real and reproducible |
| Test suite / debugger breakage | Yes | No |

The bottom line is that gevent's problems are **structural** — they can blow up at import time in ways that are hard to predict and impossible to reason about across a full dependency tree. Free-threaded `gthread`'s problems are **concurrency bugs** — they require true parallel access to shared mutable state to trigger, and they can be systematically found and fixed with locks, audits, and proper session scoping.

---

## Part 3: Migrating a Gevent-Monkeypatched Flask Dockerfile to Free-Threaded gthread

### The 5-step upgrade

---

### Step 1 — Remove gevent from your dependencies

```diff
# requirements.txt  (or pyproject.toml)
- gevent
- gunicorn[gevent]    # if you pinned this variant
+ gunicorn             # plain gunicorn is enough
```

---

### Step 2 — Replace your gunicorn launch command

```diff
# wherever you start gunicorn (CMD, entrypoint script, Procfile, etc.)
- gunicorn app:app --worker-class gevent --worker-connections 1000 --workers 2
+ python3.14t -m gunicorn app:app --worker-class gthread --workers 2 --threads 4
```

The formula for `--threads` is your CPU count (`nproc` at runtime).
`workers × threads` should equal `(2 × cores) + 1` as usual.

---

### Step 3 — Replace the Dockerfile

> **Note:** Docker Hub does not publish official `python:3.14t-*` images.
> The cleanest workaround is letting **uv** install the free-threaded interpreter
> inside any standard Debian slim base.

```dockerfile
# syntax=docker/dockerfile:1
FROM debian:bookworm-slim AS build

# ── install uv ──────────────────────────────────────────────────────────────
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# ── install free-threaded Python 3.14t via uv ────────────────────────────────
ENV UV_PYTHON_INSTALL_DIR=/usr/local \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

RUN uv python install 3.14t

WORKDIR /app

# ── install dependencies (cached separately from app code) ───────────────────
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --python 3.14t --frozen --no-install-project --no-dev

# ── copy application code ────────────────────────────────────────────────────
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --python 3.14t --frozen --no-dev

# ── runtime stage ─────────────────────────────────────────────────────────────
FROM debian:bookworm-slim

COPY --from=build /usr/local /usr/local
COPY --from=build /app /app

WORKDIR /app

# Confirm GIL is off at container startup — fail fast if a dep re-enables it
RUN python3.14t -c \
    "import sys; assert not sys._is_gil_enabled(), 'GIL was re-enabled by a dependency!'"

EXPOSE 8000

CMD ["python3.14t", "-m", "gunicorn", \
     "--worker-class", "gthread", \
     "--workers", "2", \
     "--threads", "4", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "60", \
     "app:app"]
```

---

### Step 4 — Remove monkeypatch call from your app code

```diff
# app.py  (or wherever you patched)
- from gevent import monkey
- monkey.patch_all()

  from flask import Flask
  app = Flask(__name__)
  ...
```

Also remove any gevent-specific imports elsewhere:
- `from gevent.pool import Pool` → use `concurrent.futures.ThreadPoolExecutor`
- `from gevent.lock import BoundedSemaphore` → use `threading.Semaphore`
- `gevent.sleep()` → use `time.sleep()` (standard sleep releases the GIL natively)

---

### Step 5 — Guard shared mutable state with locks

With real thread parallelism, anything shared across requests that gevent's
cooperative scheduling previously protected by accident now needs an explicit lock.

**Common patterns to fix:**

```python
# ❌ Before — gevent's cooperative scheduling made this "accidentally safe"
_cache = {}

def get_data(key):
    if key not in _cache:
        _cache[key] = fetch_from_db(key)   # race: two threads both miss and write
    return _cache[key]


# ✅ After — explicit lock
import threading
_cache = {}
_cache_lock = threading.Lock()

def get_data(key):
    with _cache_lock:
        if key not in _cache:
            _cache[key] = fetch_from_db(key)
    return _cache[key]
```

```python
# ❌ Before — module-level counter was "safe" under gevent
request_count = 0

@app.route("/")
def index():
    global request_count
    request_count += 1    # not atomic under real threads


# ✅ After
import threading
_count_lock = threading.Lock()
request_count = 0

@app.route("/")
def index():
    global request_count
    with _count_lock:
        request_count += 1
```

Per-request state (Flask's `g`, `request`, `session`) is already thread-safe and needs no changes.

---

### Quick verification

```bash
# Build
docker build -t myapp .

# Verify GIL is off inside the container
docker run --rm myapp python3.14t -c \
  "import sys; print('GIL disabled:', not sys._is_gil_enabled())"
# → GIL disabled: True

# Run
docker run -p 8000:8000 myapp
```

---

### What to watch after deploying

| Signal | Likely cause | Fix |
|---|---|---|
| `GIL disabled: False` at startup | A C extension re-enabled the GIL | Run the dep audit script; set `PYTHON_GIL=0` to force-keep it off (only if you've verified the ext is safe) |
| Intermittent wrong values in a counter or cache | Unprotected shared state | Add `threading.Lock()` around the mutation |
| DB connection pool timeouts under load | Pool sized for old worker count | Increase `pool_size` to match `workers × threads` |
| `RecursionError` on startup | You left `monkey.patch_all()` in the code | Remove it (Step 4 above) |

---

## Part 4: Benchmark Suite — gevent vs gthread/3.14t

### File layout

```
bench/
├── app.py                    ← Flask app under test (identical for both runs)
├── gunicorn_gevent.conf.py   ← gevent worker config
├── gunicorn_gthread.conf.py  ← gthread worker config (run with python3.14t)
└── bench.py                  ← drives both servers and prints comparison table
```

### Prerequisites

```bash
# Standard Python (any recent version) for the gevent run
pip install flask gunicorn gevent requests

# Free-threaded Python 3.14t
pyenv install 3.14t && pyenv local 3.14t
# — or —
uv python install 3.14t

# Install deps under 3.14t as well
python3.14t -m pip install flask gunicorn requests
```

### Run

```bash
cd bench/
python bench.py

# Flags
python bench.py --concurrency 50 --requests 500   # more aggressive
python bench.py --no-gthread                       # gevent only
python bench.py --concurrency 5 --requests 50 --warmup 5  # quick smoke test
```

---

### `app.py` — Flask app under test

```python
"""
app.py — the Flask application used for both benchmark runs.

Identical code is served under both worker types so any performance
difference is purely the server/interpreter, not the application.

Routes
------
/io   simulated I/O wait (e.g. a slow DB query or upstream API call)
/cpu  CPU-bound work: count primes up to N
/mix  I/O wait followed by a small CPU burst (realistic endpoint shape)
"""

import time
import math
from flask import Flask, jsonify

app = Flask(__name__)

# ── helpers ──────────────────────────────────────────────────────────────────

def _count_primes(limit: int) -> int:
    """Pure-Python prime count. Deliberately un-optimised to burn CPU."""
    count = 0
    for n in range(2, limit):
        for i in range(2, int(math.isqrt(n)) + 1):
            if n % i == 0:
                break
        else:
            count += 1
    return count


# ── routes ───────────────────────────────────────────────────────────────────

@app.route("/io")
def io_bound():
    """
    Simulates a 50 ms I/O wait (DB query, cache miss, upstream call).

    Under gevent: the greenlet yields during sleep so other greenlets run.
    Under gthread: the OS thread blocks but other threads continue freely.
    Both handle high concurrency well here; this is the classic gevent win.
    """
    time.sleep(0.05)
    return jsonify({"endpoint": "io", "slept_ms": 50})


@app.route("/cpu")
def cpu_bound():
    """
    Counts primes up to 8 000.

    Under gevent:  only one greenlet runs at a time (no real parallelism).
    Under gthread + python3.14t: threads run on separate cores simultaneously.
    This is the endpoint where the GIL removal makes the biggest difference.
    """
    result = _count_primes(8_000)
    return jsonify({"endpoint": "cpu", "primes_below_8000": result})


@app.route("/mix")
def mixed():
    """
    30 ms I/O wait then a small CPU burst (primes to 4 000).
    Closest to a real-world endpoint that does some DB work then computation.
    """
    time.sleep(0.03)
    result = _count_primes(4_000)
    return jsonify({"endpoint": "mix", "primes": result})


@app.route("/health")
def health():
    return jsonify({"status": "ok"})


if __name__ == "__main__":
    # Dev only — not used by the benchmark (gunicorn serves it)
    app.run(debug=False)
```

---

### `gunicorn_gevent.conf.py`

```python
"""
gunicorn_gevent.conf.py — gevent monkeypatch config.

Start with:
    gunicorn -c gunicorn_gevent.conf.py app:app
"""

import multiprocessing

# Gevent: one process per core, many greenlets per process
workers = multiprocessing.cpu_count()
worker_class = "gevent"
worker_connections = 200        # greenlets per worker

bind = "127.0.0.1:8001"
timeout = 60
accesslog = "-"
errorlog = "-"
loglevel = "warning"            # keep output clean during benchmark
```

---

### `gunicorn_gthread.conf.py`

```python
"""
gunicorn_gthread.conf.py — free-threaded gthread config.

Start with:
    python3.14t -m gunicorn -c gunicorn_gthread.conf.py app:app

The GIL check will abort startup if a dependency silently re-enables it.
"""

import multiprocessing
import sys

# Hard-fail if we're not actually running free-threaded.
# Remove this check if you want to run the same config under standard Python
# as a sanity comparison.
if sys._is_gil_enabled():
    raise RuntimeError(
        "\n\nGIL is ENABLED. Run this config with python3.14t, not python.\n"
        "If you installed a C extension that re-enabled the GIL, set\n"
        "PYTHON_GIL=0 in the environment (only after verifying it is safe).\n"
    )

cores = multiprocessing.cpu_count()

# Fewer workers, more threads — threads are genuinely parallel under 3.14t.
# Total concurrency = workers × threads, aim for (2×cores)+1 total.
workers = max(2, cores // 2)
threads = cores
worker_class = "gthread"

bind = "127.0.0.1:8002"
timeout = 60
accesslog = "-"
errorlog = "-"
loglevel = "warning"
```

---

### `bench.py` — benchmark driver

```python
#!/usr/bin/env python3
"""
bench.py — drives both gunicorn servers and prints a side-by-side comparison.

Usage:
    python bench.py [--concurrency N] [--requests N] [--no-gevent] [--no-gthread]

Requirements:
    pip install requests
    # gevent server also needs: pip install gevent
    # gthread server needs python3.14t in PATH
"""

import argparse
import subprocess
import sys
import time
import threading
import statistics
import shutil
from dataclasses import dataclass, field
from typing import Optional

try:
    import requests
except ImportError:
    sys.exit("requests is required: pip install requests")


# ── CLI ──────────────────────────────────────────────────────────────────────

def parse_args():
    p = argparse.ArgumentParser(description="gevent vs gthread Flask benchmark")
    p.add_argument("--concurrency", type=int, default=20,
                   help="concurrent threads hammering each endpoint (default 20)")
    p.add_argument("--requests", type=int, default=200,
                   help="total requests per endpoint per server (default 200)")
    p.add_argument("--warmup", type=int, default=20,
                   help="warmup requests before timing starts (default 20)")
    p.add_argument("--no-gevent", action="store_true",
                   help="skip the gevent run")
    p.add_argument("--no-gthread", action="store_true",
                   help="skip the gthread run")
    return p.parse_args()


# ── server management ────────────────────────────────────────────────────────

def start_server(cmd: list[str], port: int, name: str) -> subprocess.Popen:
    """Start a gunicorn process and wait until it is accepting connections."""
    print(f"\n▶  Starting {name} on port {port}...")
    proc = subprocess.Popen(
        cmd,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
    )

    deadline = time.time() + 15
    while time.time() < deadline:
        try:
            requests.get(f"http://127.0.0.1:{port}/health", timeout=1)
            print(f"   {name} is up  (pid {proc.pid})")
            return proc
        except Exception:
            time.sleep(0.2)

    proc.terminate()
    raise RuntimeError(f"{name} did not start within 15 s — check dependencies")


def stop_server(proc: subprocess.Popen, name: str):
    proc.terminate()
    try:
        proc.wait(timeout=5)
    except subprocess.TimeoutExpired:
        proc.kill()
    print(f"■  {name} stopped")


# ── load generation ───────────────────────────────────────────────────────────

@dataclass
class EndpointResult:
    name: str
    total: int = 0
    errors: int = 0
    latencies: list[float] = field(default_factory=list)

    @property
    def success(self): return self.total - self.errors

    @property
    def p50(self): return statistics.median(self.latencies) * 1000 if self.latencies else 0

    @property
    def p95(self):
        if not self.latencies:
            return 0
        s = sorted(self.latencies)
        idx = int(len(s) * 0.95)
        return s[min(idx, len(s) - 1)] * 1000

    @property
    def rps(self): return self.success / self.elapsed if self.elapsed else 0

    elapsed: float = 0.0


def _worker(url: str, n: int, results: EndpointResult, lock: threading.Lock):
    session = requests.Session()
    for _ in range(n):
        t0 = time.perf_counter()
        try:
            r = session.get(url, timeout=10)
            elapsed = time.perf_counter() - t0
            with lock:
                results.total += 1
                if r.status_code != 200:
                    results.errors += 1
                else:
                    results.latencies.append(elapsed)
        except Exception:
            elapsed = time.perf_counter() - t0
            with lock:
                results.total += 1
                results.errors += 1


def run_endpoint(
    base_url: str,
    path: str,
    concurrency: int,
    total_requests: int,
    warmup: int,
) -> EndpointResult:
    url = base_url + path
    result = EndpointResult(name=path)
    lock = threading.Lock()

    # warmup
    for _ in range(warmup):
        try:
            requests.get(url, timeout=5)
        except Exception:
            pass

    # timed load
    per_thread = max(1, total_requests // concurrency)
    threads = [
        threading.Thread(target=_worker, args=(url, per_thread, result, lock))
        for _ in range(concurrency)
    ]

    t0 = time.perf_counter()
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    result.elapsed = time.perf_counter() - t0

    return result


# ── reporting ─────────────────────────────────────────────────────────────────

def fmt(val: float, unit: str = "") -> str:
    return f"{val:7.1f}{unit}"


def print_table(gevent_results: Optional[dict], gthread_results: Optional[dict]):
    endpoints = ["/io", "/cpu", "/mix"]
    col = 18

    bar = "─" * (col * 5 + 3)
    print(f"\n{'─'*80}")
    print("  BENCHMARK RESULTS")
    print(f"{'─'*80}\n")

    header = (
        f"  {'Endpoint':<10}"
        f"  {'Metric':<14}"
        f"  {'gevent':>{col}}"
        f"  {'gthread/3.14t':>{col}}"
        f"  {'Δ (gthread vs gevent)':>{col}}"
    )
    print(header)
    print(f"  {bar}")

    for ep in endpoints:
        g = gevent_results.get(ep) if gevent_results else None
        t = gthread_results.get(ep) if gthread_results else None

        def row(label, gval, tval, unit="", higher_better=True):
            g_str = fmt(gval, unit) if gval is not None else "  (skipped)"
            t_str = fmt(tval, unit) if tval is not None else "  (skipped)"

            if gval is not None and tval is not None and gval > 0:
                diff_pct = (tval - gval) / gval * 100
                sign = "+" if diff_pct >= 0 else ""
                winner = "✓" if (diff_pct >= 0) == higher_better else "✗"
                d_str = f"{winner} {sign}{diff_pct:5.1f}%"
            else:
                d_str = "        n/a"

            print(
                f"  {ep if label == 'RPS' else '':10}"
                f"  {label:<14}"
                f"  {g_str:>{col}}"
                f"  {t_str:>{col}}"
                f"  {d_str:>{col}}"
            )

        row("RPS",   g.rps  if g else None, t.rps  if t else None, " req/s", higher_better=True)
        row("p50",   g.p50  if g else None, t.p50  if t else None, " ms",    higher_better=False)
        row("p95",   g.p95  if g else None, t.p95  if t else None, " ms",    higher_better=False)
        row("Errors",g.errors if g else None, t.errors if t else None, "",   higher_better=False)
        print(f"  {bar}")

    print()
    print("  ✓ = gthread/3.14t is better   ✗ = gevent is better")
    print("  Higher RPS is better. Lower latency (ms) is better.")
    print()


# ── main ──────────────────────────────────────────────────────────────────────

ENDPOINTS = ["/io", "/cpu", "/mix"]

def benchmark_server(base_url: str, label: str, concurrency: int,
                     total: int, warmup: int) -> dict:
    results = {}
    for ep in ENDPOINTS:
        print(f"   {label}  {ep} ... ", end="", flush=True)
        r = run_endpoint(base_url, ep, concurrency, total, warmup)
        results[ep] = r
        print(f"{r.rps:.1f} req/s  p50={r.p50:.0f}ms  errors={r.errors}/{r.total}")
    return results


def main():
    args = parse_args()

    # Locate python3.14t
    py314t = shutil.which("python3.14t")
    if not args.no_gthread and py314t is None:
        print("⚠  python3.14t not found in PATH — skipping gthread run.")
        print("   Install with:  pyenv install 3.14t  or  uv python install 3.14t")
        args.no_gthread = True

    print(f"\nBenchmark settings:")
    print(f"  concurrency : {args.concurrency} threads")
    print(f"  requests    : {args.requests} per endpoint")
    print(f"  warmup      : {args.warmup} requests")

    gevent_results = None
    gthread_results = None

    # ── gevent run ────────────────────────────────────────────────────────────
    if not args.no_gevent:
        try:
            import gevent  # noqa: F401
        except ImportError:
            print("\n⚠  gevent not installed — skipping gevent run.")
            print("   pip install gevent")
            args.no_gevent = True

    if not args.no_gevent:
        cmd = [
            sys.executable, "-m", "gunicorn",
            "-c", "gunicorn_gevent.conf.py",
            "app:app",
        ]
        proc = start_server(cmd, 8001, "gevent")
        try:
            gevent_results = benchmark_server(
                "http://127.0.0.1:8001", "gevent  ",
                args.concurrency, args.requests, args.warmup,
            )
        finally:
            stop_server(proc, "gevent")

    # ── gthread run ───────────────────────────────────────────────────────────
    if not args.no_gthread:
        cmd = [
            py314t, "-m", "gunicorn",
            "-c", "gunicorn_gthread.conf.py",
            "app:app",
        ]
        proc = start_server(cmd, 8002, "gthread/3.14t")
        try:
            gthread_results = benchmark_server(
                "http://127.0.0.1:8002", "gthread ",
                args.concurrency, args.requests, args.warmup,
            )
        finally:
            stop_server(proc, "gthread/3.14t")

    # ── results ───────────────────────────────────────────────────────────────
    if gevent_results or gthread_results:
        print_table(gevent_results, gthread_results)
    else:
        print("\nNo results — both runs were skipped.")


if __name__ == "__main__":
    main()
```

---

### Expected results on a multi-core machine

`/io` — likely a draw, or gevent slightly ahead. Both handle 20 concurrent sleepers fine; gevent's greenlets are marginally cheaper to context-switch than OS threads.

`/cpu` — gthread/3.14t should win substantially (2–4× depending on core count). Under gevent only one greenlet ever runs CPU work at a time. Under `python3.14t` threads actually run on multiple cores simultaneously.

`/mix` — gthread/3.14t should win moderately. The I/O portion closes the gap; the CPU portion opens it back up.
