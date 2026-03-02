---
type: blog
title: Kida — A Template Engine Built for Free-Threaded Python
date: '2026-03-02'
series:
  id: nogil
  name: "Free-Threading in the Bengal Ecosystem"
  part: 2
  total: 6
tags:
- kida
- python
- free-threading
- templates
- jinja
category: technology
description: How Kida achieves thread-safe template rendering with copy-on-write, immutable AST, and ContextVar — techniques for the free-threading ecosystem
params:
  author: kida
---

# Kida — A Template Engine Built for Free-Threaded Python

Template engines in Python have a problem: they're often built around shared mutable state. Filters, tests, and globals live in dicts that get mutated when you call `add_filter()` or `add_global()`. The compiled template might cache things. When you throw multiple threads at it — especially under free-threaded Python where the GIL no longer serializes access — you get races.

Kida is a Jinja-like template engine for Python 3.14t, designed from the ground up for free-threading. It compiles templates to Python AST directly (no string generation), uses copy-on-write for configuration, and isolates per-render state in `ContextVar`.

---

## Run it with free-threaded Python

```bash
uv python install 3.14t
uv run --python=3.14t python -c "
from kida import Environment
env = Environment()
t = env.from_string('Hello, {{ name }}!')
print(t.render(name='World'))
"
```

Kida declares itself GIL-independent via `_Py_mod_gil = 0` (PEP 703). All public APIs are thread-safe. Run your app with `PYTHON_GIL=0` and multiple threads can render the same template concurrently.

---

## 1. Copy-on-write for filters, tests, and globals

The classic mistake: mutating a shared dict when a thread calls `add_filter()`. Under free-threading, that's a data race.

Kida uses **copy-on-write** instead. Every mutation creates a new dict and swaps it in:

```python
def add_filter(self, name: str, func: Callable[..., Any]) -> None:
    """Add a filter (copy-on-write)."""
    new_filters = self._filters.copy()
    new_filters[name] = func
    self._filters = new_filters  # Atomic swap
```

Same pattern for `add_test()`, `add_global()`, `update_filters()`, and `update_tests()`. No locks. A thread that's mid-render keeps a reference to the old dict; a thread that calls `add_filter()` gets the new one. No shared mutable state.

**Learning:** Copy-on-write scales with free-threading. The "cost" is an extra dict copy on config change — rare compared to render volume. The benefit is zero lock contention.

---

## 2. Immutable AST nodes

Kida compiles templates to a custom AST before emitting Python AST. Those nodes are frozen:

```python
@dataclass(frozen=True, slots=True)
class Node:
    """Base class for all AST nodes.
    Nodes are immutable for thread-safety.
    """
    lineno: int
    col_offset: int
```

Immutable AST → immutable compiled template. Once a template is compiled, nothing about it changes. Multiple threads can render it concurrently without touching shared mutable structure.

**Learning:** If your pipeline produces intermediate structures (AST, IR, compiled output), make them immutable. It eliminates a whole class of races.

---

## 3. ContextVar for per-render state

Template engines need internal state during render: current template name, line number, include depth, cached blocks. The old pattern: inject `_template`, `_line`, `_include_depth` into the user's context dict. That pollutes the user namespace and risks key collisions.

Kida uses `ContextVar` instead:

```python
# Per-render state isolated from user context
_render_context: ContextVar[RenderContext | None] = ContextVar(
    "kida_render_context", default=None
)
```

`RenderContext` holds template name, line, include depth, block cache, and template stack. Each render sets the ContextVar at the start and clears it at the end. User context stays clean. No `_`-prefixed keys.

**Learning:** ContextVar is for "state that belongs to this logical task." It propagates to `asyncio.to_thread()` in Python 3.14, so async rendering works without special handling.

---

## 4. StringBuilder for render throughput

Kida generates two render modes from one template: `render()` (single string) and `render_stream()` (generator). The fast path uses a list + `''.join()`:

```python
# Generated code for render()
def render(ctx, _blocks=None):
    buf = []
    _append = buf.append
    _append("Hello, ")
    _append(_e(_s(ctx["name"])))
    return ''.join(buf)
```

That's O(n) vs O(n²) for repeated string concatenation. Each `render()` call creates its own `buf` — local state only. No shared buffers.

**Learning:** Avoid shared mutable buffers. Per-call local state is thread-safe by construction.

---

## 5. Compute outside the lock

The template LRU cache uses a lock for the dict, but the expensive part — compiling a template — happens *outside* the lock:

```python
# Miss - compute value
# Compute outside lock to avoid blocking other threads
value = factory(key) if pass_key else factory()

# Store result
with self._lock:
    if key in self._cache:
        return self._cache[key]  # Another thread beat us
    self._cache[key] = value
    return value
```

If two threads miss the same key, both might compute. The second one discards its result. The alternative — holding the lock during compilation — would serialize all cache misses. Under free-threading, that defeats parallelism.

**Learning:** Minimize lock scope. Compute first, then lock only to store. Accept rare duplicate work for better throughput.

---

## 6. Atomic bytecode cache writes

The disk cache writes compiled bytecode. A partial write (crash mid-write) would corrupt the cache. Kida uses the atomic rename pattern:

```python
tmp_path = path.with_suffix(".tmp")
with open(tmp_path, "wb") as f:
    marshal.dump(code, f)
tmp_path.rename(path)  # Atomic on POSIX
```

Write to temp, then rename. The rename is atomic. Readers never see a half-written file.

**Learning:** For file caches, write-then-rename is the standard pattern. No locks, no corruption.

---

## 7. Declaring GIL independence (PEP 703)

Kida explicitly declares that it doesn't rely on the GIL for correctness:

```python
def __getattr__(name: str) -> object:
    if name == "_Py_mod_gil":
        # Signal: this module is safe for free-threading
        # 0 = Py_MOD_GIL_NOT_USED
        return 0
    raise AttributeError(...)
```

`_Py_mod_gil = 0` (Py_MOD_GIL_NOT_USED) tells the runtime and tooling that this module is designed for free-threading. It's a promise, not a guarantee — but it signals intent to the ecosystem.

---

## What this means in practice

On free-threaded Python 3.14t, multiple threads can render Kida templates concurrently without GIL contention. Benchmarks show ~1.76ms (Kida) vs ~1.97ms (Jinja2) for 8-thread concurrent rendering. The architecture — copy-on-write, immutable AST, ContextVar, compute-outside-lock — pays off when threads run in parallel.

Kida is part of the Bengal ecosystem (Chirp, Pounce, Patitas, Rosettes) — a stack built for Python's free-threaded future.

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Kida documentation](https://lbliii.github.io/kida/)
