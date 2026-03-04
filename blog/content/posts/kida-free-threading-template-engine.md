---
type: blog
title: "Kida — A Template Engine Built for Free-Threaded Python"
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
- jinja-alternative
- python-3.14
- nogil
- template-engine
category: technology
description: "How Kida achieves thread-safe template rendering with copy-on-write, immutable AST, and ContextVar — benchmark data and patterns for the free-threading ecosystem"
params:
  author: kida
---

# Kida — A Template Engine Built for Free-Threaded Python

Template engines in Python are built around shared mutable state. Filters, tests, and globals live in dicts that get mutated when you call `add_filter()` or `add_global()`. The compiled template caches things. When you throw multiple threads at that — especially under free-threaded Python where the GIL no longer serializes access — you get races.

Kida is a Jinja-compatible template engine for Python 3.14t. If you know Jinja2, you know Kida's syntax. The difference is under the hood: Kida compiles templates to Python AST directly (no string generation), uses copy-on-write for configuration, and isolates per-render state in `ContextVar`.

---

:::{tip} Series context
This is **Part 2 of 6** in *Free-Threading in the Bengal Ecosystem*. Kida is the template layer — used by both [Bengal](/blog/posts/bengal-free-threading-architecture/) (SSG) and [Chirp](/blog/posts/chirp-free-threading-web-framework/) (web framework).
:::

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

Kida declares itself GIL-independent via `_Py_mod_gil = 0` (PEP 703). All public APIs are thread-safe.

---

## Performance

Kida's benchmark suite (`benchmarks/`) compares against Jinja2 across template complexity and concurrency levels.

:::{list-table} Single-threaded rendering
:header-rows: 1

- - Template
  - Kida
  - Jinja2
  - Speedup
- - Minimal (variable output)
  - 8.25 µs
  - 11.97 µs
  - 1.45x
- - Small (conditionals + loops)
  - 17.03 µs
  - 27.69 µs
  - 1.63x
- - Large (complex layout)
  - 1.91 ms
  - 4.09 ms
  - 2.14x
:::

:::{list-table} Concurrent rendering (100 renders, free-threaded 3.14t)
:header-rows: 1

- - Workers
  - Kida
  - Jinja2
  - Advantage
- - 1
  - 1.80 ms
  - 1.80 ms
  - ~same
- - 4
  - 1.62 ms
  - 1.90 ms
  - 1.17x
- - 8
  - 1.76 ms
  - 1.97 ms
  - 1.12x
:::

The single-threaded advantage comes from AST-native compilation — Kida generates `ast.Module` directly, not source strings. The concurrent advantage is smaller because Jinja2 happens to be reasonably well-behaved under threading for read-only renders, but Kida's architecture means it *stays* safe when you mutate config.

---

## Copy-on-write for filters, tests, and globals

The classic mistake: mutating a shared dict when a thread calls `add_filter()`. Under free-threading, that's a data race.

Kida uses **copy-on-write** instead. Every mutation creates a new dict and swaps it in:

```python
def add_filter(self, name: str, func: Callable[..., Any]) -> None:
    new_filters = self._filters.copy()
    new_filters[name] = func
    self._filters = new_filters  # Atomic swap
```

Same pattern for `add_test()`, `add_global()`, `update_filters()`, and `update_tests()`. No locks. A thread mid-render keeps a reference to the old dict; a thread that calls `add_filter()` gets the new one.

:::{tab-set}
:::{tab-item} Jinja2 (shared mutable dict)
```python
env.filters["my_filter"] = my_func
```
Direct dict mutation. Under free-threading, if another thread is iterating this dict during rendering, you get a race.
:::
:::{tab-item} Kida (copy-on-write)
```python
env.add_filter("my_filter", my_func)
```
Creates a new dict, copies all entries, adds the new one, swaps atomically. Readers see the old or the new — never a torn state.
:::
:::

:::{tip} Pattern
Copy-on-write scales with free-threading. The "cost" is an extra dict copy on config change — rare compared to render volume. The benefit is zero lock contention on the hot path.
:::

---

## Immutable AST nodes

Kida compiles templates to a custom AST before emitting Python AST. Those nodes are frozen:

```python
@dataclass(frozen=True, slots=True)
class Output(Node):
    """Output expression: {{ expr }}"""
    expr: Expr
    escape: bool = True
```

Immutable AST means immutable compiled templates. Once compiled, nothing changes. Multiple threads render the same template concurrently without touching shared mutable structure. This is also what enables Kida's fragment caching — frozen nodes can be stored and reused safely.

---

## ContextVar for per-render state

Template engines need internal state during render: current template name, line number, include depth, cached blocks. The old pattern: inject `_template`, `_line`, `_include_depth` into the user's context dict. That pollutes the user namespace and risks key collisions.

Kida uses `ContextVar` instead:

```python
_render_context: ContextVar[RenderContext | None] = ContextVar(
    "render_context", default=None
)
```

`RenderContext` holds template name, line, include depth, block cache, and template stack. Each render sets it at entry and clears it at exit. User context stays clean — no `_`-prefixed keys leaking into template variables.

:::{tip} Pattern
`ContextVar` is for "state that belongs to this logical task." It propagates to `asyncio.to_thread()` in Python 3.14, so async rendering works without special handling.
:::

---

## StringBuilder for render throughput

Kida generates two render modes from one template: `render()` (single string) and `render_stream()` (generator). The fast path uses a list + `''.join()`:

```python
def render(ctx, _blocks=None):
    buf = []
    _append = buf.append
    _append("Hello, ")
    _append(_e(_s(ctx["name"])))
    return ''.join(buf)
```

That's O(n) vs O(n²) for repeated string concatenation. Each `render()` call creates its own `buf` — local state only, no shared buffers.

---

## Compute outside the lock

The template LRU cache uses a lock for the dict, but the expensive part — compiling a template — happens *outside* the lock:

```python
# Compute outside lock to avoid blocking other threads
value = factory(key) if pass_key else factory()

# Store result
with self._lock:
    if key in self._cache:
        return self._cache[key]  # Another thread beat us
    self._cache[key] = value
    return value
```

If two threads miss the same key, both compile. The second one discards its result. The alternative — holding the lock during compilation — would serialize all cache misses.

:::{tip} Pattern
Minimize lock scope. Compute first, then lock only to store. Accept rare duplicate work for better throughput under contention.
:::

---

## Atomic bytecode cache writes

The disk cache writes compiled bytecode. A partial write (crash mid-write) would corrupt the cache. Kida uses the atomic rename pattern:

```python
tmp_path = path.with_suffix(".tmp")
with open(tmp_path, "wb") as f:
    marshal.dump(code, f)
tmp_path.rename(path)  # Atomic on POSIX
```

Write to temp, then rename. Readers never see a half-written file. No locks needed for file-level safety.

---

## What this means in practice

On free-threaded Python 3.14t, multiple threads render Kida templates concurrently without GIL contention. The architecture — copy-on-write config, immutable AST, ContextVar isolation, compute-outside-lock caching — pays off when threads run in parallel. But even on GIL builds, these patterns eliminate shared mutable state bugs that would otherwise surface as subtle, hard-to-reproduce issues.

Kida powers templates in both [Bengal](/blog/posts/bengal-free-threading-architecture/) (static site generation) and [Chirp](/blog/posts/chirp-free-threading-web-framework/) (web serving). It's the rendering layer in a stack built for Python's free-threaded future.

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Kida documentation](https://lbliii.github.io/kida/)
- [Kida source](https://github.com/lbliii/kida)
- **Next in series:** [Patitas — A Markdown Parser Built for Parallel Parsing](/blog/posts/patitas-free-threading-markdown-parser/)
