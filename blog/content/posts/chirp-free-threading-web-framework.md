---
type: blog
title: Chirp — A Web Framework Built for Free-Threaded Python
date: '2026-03-02'
series:
  id: nogil
  name: "Free-Threading in the Bengal Ecosystem"
  part: 6
  total: 6
tags:
- chirp
- python
- free-threading
- web-framework
- asgi
category: technology
description: How Chirp achieves thread-safe request handling with ContextVar, frozen config, double-check freeze locking, and per-worker lifecycle — techniques for the free-threading ecosystem
params:
  author: kida
---

# Chirp — A Web Framework Built for Free-Threaded Python

Web frameworks coordinate a lot of shared state: routes, middleware, template filters, session storage, request context. Under free-threaded Python, multiple worker threads can hit the same app instance concurrently. If the app mutates shared dicts during setup, or if request-scoped data leaks across threads, you get races. The framework must either isolate state per request or protect shared state with locks.

Chirp is a Python web framework for the modern web platform — HTML fragments, streaming, SSE, htmx. It targets Python 3.14t and declares free-threading support via PEP 703. It runs on Pounce (thread-based workers on nogil, process-based on GIL). Here's how it stays safe.

---

## Run it with free-threaded Python

```bash
uv python install 3.14t
uv run --python=3.14t python -c "
from chirp import App, AppConfig

app = App(AppConfig())
@app.route('/')
def index():
    return 'Hello, World!'
app.run()
"
```

Or with Pounce for multi-worker production:

```bash
uv run --python=3.14t pounce myapp:app --workers 4
```

On Python 3.14t, Pounce runs four worker threads. Each thread shares the same Chirp app instance. Chirp's design ensures that sharing is safe.

---

## 1. PEP 703 declaration

Chirp declares itself GIL-independent at module load:

```python
# chirp/__init__.py
_Py_mod_gil = 0
```

This tells the interpreter that Chirp does not rely on the GIL for correctness. The rest of the framework backs that claim: request-scoped state in `ContextVar`, frozen config, and explicit locks where shared mutable state exists.

---

## 2. Mutable setup, frozen runtime

The app is mutable during setup (decorators, `add_middleware`, `mount_pages`). At runtime — when the first request arrives or lifespan starts — it freezes. Routes, middleware, and template env become immutable. No more `@app.route()` after that.

Under free-threading, multiple worker threads can call `__call__()` concurrently on the first request. Chirp uses double-check locking so exactly one thread compiles the app:

```python
def _ensure_frozen(self) -> None:
    """Thread-safe freeze with double-check locking.

    Under free-threading (3.14t), multiple ASGI worker threads could
    call __call__() concurrently on first request.
    """
    if self._frozen:
        return
    with self._freeze_lock:
        if self._frozen:
            return
        self._freeze()
```

The fast path (`if self._frozen`) avoids lock acquisition on every request. The lock ensures only one thread runs `_freeze()`. After freeze, `_router`, `_middleware`, and `_kida_env` are set and never mutated.

**Learning:** Double-check locking for one-time initialization. Fast path for the common case; lock for the transition.

---

## 3. Frozen config

`AppConfig` and `SessionConfig` are frozen dataclasses:

```python
@dataclass(frozen=True, slots=True)
class AppConfig:
    """Application configuration. Immutable after creation."""
    host: str = "127.0.0.1"
    port: int = 8000
    debug: bool = False
    template_dir: str | Path = "templates"
    # ... 50+ fields, all immutable
```

Config is created once and passed around. Workers read `self.config.port`, `self.config.debug` — no locks, no races.

---

## 4. ContextVar for request-scoped state

Request, session, and `g` (request-scoped namespace) live in `ContextVar`:

```python
# chirp/context.py
request_var: ContextVar[Request] = ContextVar("chirp_request")

class _RequestGlobals:
    """Request-scoped namespace (Flask's g)."""
    def __init__(self) -> None:
        object.__setattr__(self, "_store", ContextVar("chirp_g", default=None))
```

```python
# chirp/middleware/sessions.py
_session_var: ContextVar[dict[str, Any] | None] = ContextVar("chirp_session", default=None)
```

`ContextVar` is task-local under asyncio and thread-local under free-threading. Each request gets its own context. No shared mutable state between requests.

**Learning:** `ContextVar` for per-request or per-task state. No locks needed — the runtime isolates contexts.

---

## 5. Per-worker lifecycle for async resources

httpx clients, DB connection pools, and other async resources bind to an event loop. Under Pounce's thread workers, each worker has its own loop. Chirp provides per-worker hooks:

```python
_client_var: ContextVar[httpx.AsyncClient | None] = ContextVar("hn_client", default=None)

@app.on_worker_startup
async def create_client():
    _client_var.set(httpx.AsyncClient(...))

@app.on_worker_shutdown
async def close_client():
    client = _client_var.get(None)
    if client:
        await client.aclose()
```

Pounce sends `pounce.worker.startup` and `pounce.worker.shutdown` scopes. Chirp runs the hooks on each worker's event loop. Each worker gets its own client; no sharing across threads.

**Learning:** Per-worker `ContextVar` + lifecycle hooks for async resources. One client per worker, no cross-thread reuse.

---

## 6. Lock for shared mutable state

When app code *does* share mutable state — caches, counters, in-memory stores — use a lock. The Hackernews and dashboard examples show the pattern:

```python
# Shared mutable state (dashboard example)
_readings: dict[str, SensorReading] = {}
_lock = threading.Lock()

def _update_sensor(sensor_id: str) -> SensorReading:
    reading = _make_reading(sensor_id)
    with _lock:
        _readings[sensor_id] = reading
    return reading
```

Data models are frozen (`@dataclass(frozen=True, slots=True)`). The dict holding them is mutable. The lock protects the dict.

**Learning:** Frozen dataclasses for values; lock for the container. Minimize what the lock protects.

---

## 7. ToolEventBus — Lock for subscriber set

The MCP tool event bus broadcasts `ToolCallEvent` to SSE subscribers. Events are frozen; the subscriber set is mutable:

```python
@dataclass(frozen=True, slots=True)
class ToolCallEvent:
    tool_name: str
    arguments: dict[str, Any]
    result: Any
    timestamp: float
    call_id: str

class ToolEventBus:
    def __init__(self) -> None:
        self._subscribers: set[asyncio.Queue[ToolCallEvent | None]] = set()
        self._lock = threading.Lock()

    async def emit(self, event: ToolCallEvent) -> None:
        with self._lock:
            subscribers = set(self._subscribers)
        for queue in subscribers:
            queue.put_nowait(event)
```

Each subscriber has its own queue. The lock protects add/remove of subscribers. Emit copies the set under the lock, then iterates without holding it — avoids blocking emitters on slow consumers.

---

## 8. Frozen forms and uploads

Form binding uses frozen dataclasses for type-safe, immutable values:

```python
@dataclass(frozen=True, slots=True)
class UploadFile:
    filename: str
    content_type: str
    size: int
    _content: bytes
```

`form_from(request, MyForm)` returns a frozen instance. No mutation, no shared state.

---

## What this means in practice

Chirp runs on Pounce with four worker threads. One app instance, one Kida environment, one router. Request-scoped state lives in `ContextVar`. Config and events are frozen. Shared mutable state uses locks. App authors get clear patterns: `ContextVar` for per-request, `on_worker_startup` for per-worker async resources, `Lock` for shared caches.

Chirp is the web framework in the Bengal stack — it serves pages built with Kida, Patitas, and Rosettes, running on Pounce. A full pipeline built for Python's free-threaded future.

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [PEP 703 — Making the global interpreter lock optional](https://peps.python.org/pep-0703/)
- [Chirp documentation](https://lbliii.github.io/chirp/)
