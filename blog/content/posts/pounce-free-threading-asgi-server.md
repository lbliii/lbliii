---
type: blog
title: Pounce — Thread-Based ASGI Workers with Shared Immutable Config
date: '2026-03-02'
series:
  id: nogil
  name: "Free-Threading in the Bengal Ecosystem"
  part: 5
  total: 6
tags:
- pounce
- python
- free-threading
- asgi
- server
category: technology
description: How Pounce achieves thread-based workers on free-threaded Python with shared immutable config, per-request compressors, and graceful reload — techniques for the free-threading ecosystem
params:
  author: kida
---

# Pounce — Thread-Based ASGI Workers with Shared Immutable Config

ASGI servers traditionally use process-based workers. Fork, load the app, serve requests — each process has its own interpreter and memory. Under free-threaded Python, threads become viable: one interpreter, N event loops, shared memory. No fork overhead, no IPC. But sharing state between worker threads requires care — mutable config, shared compressors, or connection counters can race.

Pounce is a free-threading-native ASGI server for Python 3.14t. N worker threads share one interpreter, one copy of your app, one set of frozen config. On GIL builds, it falls back to multi-process automatically.
---

## Run it with free-threaded Python

```bash
uv python install 3.14t
uv run --python=3.14t pounce myapp:app --workers 4
```

On Python 3.14t, workers are threads. One process, four threads, each with its own asyncio event loop. On standard Python, workers are processes — same API, same config. The supervisor detects the runtime via `sys._is_gil_enabled()` and adapts automatically.

---

## 1. Threads vs processes — one runtime, two modes

Pounce detects the GIL at startup and picks the worker model:

```python
def is_gil_enabled() -> bool:
    """Check whether the GIL is active in the current interpreter."""
    return getattr(sys, "_is_gil_enabled", lambda: True)()

def detect_worker_mode() -> WorkerMode:
    """Choose worker strategy based on GIL state."""
    return "process" if is_gil_enabled() else "thread"
```

The supervisor uses it:

```python
self._mode: WorkerMode = mode or detect_worker_mode()

# In _spawn_worker():
if self._mode == "thread":
    target = threading.Thread(target=worker.run, name=f"pounce-worker-{worker_id}")
else:
    target = multiprocessing.Process(target=worker.run, name=f"pounce-worker-{worker_id}")
target.start()
```

Same `Worker` class, same `ServerConfig`, same request flow. Only the spawning mechanism differs. On nogil: threads share memory. On GIL: processes get isolation.

**Learning:** `sys._is_gil_enabled()` is the runtime check. Design your worker to work in both modes; the supervisor handles the spawn.

---

## 2. Shared immutable config

Workers need config: host, port, timeouts, limits, compression settings. Mutating a shared dict from multiple threads is a race. Pounce uses a frozen dataclass:

```python
@dataclass(frozen=True, slots=True)
class ServerConfig:
    """Immutable server configuration.

    All settings for a pounce server instance. Created once at startup,
    shared across all worker threads (safe because frozen).
    """
    host: str = "127.0.0.1"
    port: int = 8000
    workers: int = 1
    keep_alive_timeout: float = 5.0
    request_timeout: float = 30.0
    compression: bool = True
    # ... 30+ fields, all immutable
```

Config is created once at startup and passed to every worker. No one mutates it. No locks. Workers read `self._config.port`, `self._config.compression`, etc. — all safe.

**Learning:** Frozen dataclasses for config eliminate synchronization. Create once, share freely.

---

## 3. Per-request compressor — no shared state

Compression (gzip, zstd) requires stateful compressors. Sharing one compressor across requests would be a race. Pounce creates a fresh compressor per request:

```python
class GzipCompressor:
    """Gzip compressor using stdlib zlib.

    Creates a fresh zlib compressor per instance. Each request gets its own
    GzipCompressor — no shared state.
    """
    def __init__(self, *, level: int = 6) -> None:
        self._compressor = zlib.compressobj(level, zlib.DEFLATED, 31)
```

Each response that needs compression gets a new `GzipCompressor` or `ZstdCompressor`. No shared buffers, no contention.

**Learning:** Per-request instances beat shared state. The cost of creating a compressor is small compared to the request lifetime.

---

## 4. Brotli excluded — C extensions and the GIL

Pounce supports zstd (stdlib PEP 784) and gzip (stdlib zlib). Brotli is intentionally excluded:

```python
"""Note: Brotli (br) is intentionally excluded — the ``brotli`` C extension
re-enables the GIL on Python 3.14t, defeating pounce's free-threading
architecture. Clients that only send ``Accept-Encoding: br`` will receive
uncompressed responses.
"""
```

The `brotli` C extension re-enables the GIL when used. On a free-threaded build, that would serialize all threads whenever any thread compresses. Pounce sticks to stdlib compression — zlib and `compression.zstd` work correctly under free-threading.

**Learning:** Audit C extensions. Some re-enable the GIL. Prefer stdlib or free-threading-safe libraries.

---

## 5. Lock only where it matters

Some state *does* need locks — connection IDs, rate limiters, metrics. Pounce uses locks narrowly:

```python
_id_counter = 0
_id_lock = threading.Lock()

def next_connection_id() -> int:
    """Thread-safe — uses a lock for correctness under free-threading."""
    global _id_counter
    with _id_lock:
        _id_counter += 1
        return _id_counter
```

Connection IDs must be unique and monotonic. A lock around the increment is correct and cheap. The rest of the request path — config reads, compressor creation, ASGI call — needs no locks.

**Learning:** Identify the minimal set of shared mutable state. Lock only that. Everything else: immutable or per-request.

---

## 6. Frozen lifecycle events

Observability events (connection opened, request started, response completed) are frozen dataclasses:

```python
@dataclass(frozen=True, slots=True)
class ConnectionOpened:
    """A new TCP connection was accepted."""
    connection_id: int
    worker_id: int
    client_addr: str
    client_port: int
    protocol: str  # "h1", "h2", "websocket"
    timestamp_ns: int

@dataclass(frozen=True, slots=True)
class ResponseCompleted:
    """An HTTP response was fully sent."""
    connection_id: int
    worker_id: int
    status: int
    bytes_sent: int
    duration_ms: float
    timestamp_ns: int
```

Workers produce events; an optional `LifecycleCollector` consumes them. Events are immutable — safe to pass across thread boundaries, buffer, or log. No defensive copying.

---

## 7. Graceful reload — thread mode only

In thread mode, Pounce supports zero-downtime rolling restart:

1. Spawn new workers
2. Mark old workers for draining (finish existing connections, reject new)
3. Wait for old workers to become idle
4. Shut down old workers

In process mode, workers run in separate processes — the supervisor can't control them for rolling restart. It falls back to a brief downtime restart. Graceful reload is a thread-mode benefit: shared memory lets the supervisor signal workers to drain.

---

## What this means in practice

On free-threaded Python 3.14t, `pounce myapp:app --workers 4` runs four threads. One interpreter, one app load, shared immutable config. No fork, no IPC. Compression uses stdlib zlib and `compression.zstd` — no GIL-reenabling extensions.

Pounce is the ASGI server in the Bengal ecosystem — it serves Chirp apps, which use Kida, Patitas, and Rosettes. A stack built for Python's free-threaded future.

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Pounce documentation](https://lbliii.github.io/pounce/)
