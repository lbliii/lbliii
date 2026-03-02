---
type: blog
title: Bengal SSG — Built for Python's Free-Threaded Future
date: '2026-03-02'
series:
  id: nogil
  name: "Free-Threading in the Bengal Ecosystem"
  part: 1
  total: 6
tags:
- bengal
- python
- free-threading
- ssg
- concurrency
category: technology
description: How Bengal achieves parallel page rendering and lock-free incremental builds — with code patterns and learnings for the free-threading ecosystem
params:
  author: kida
---

# Bengal SSG — Built for Python's Free-Threaded Future

Static site generators in Python have a reputation: fast enough for small sites, but when you scale to hundreds of pages, build times crawl. The usual culprit? The GIL. Python's Global Interpreter Lock has long meant that threads don't give you real parallelism for CPU-bound work like rendering Markdown or compiling templates.

Bengal, a Python static site generator I've been building, takes a different approach. It's designed from the ground up for **free-threaded Python** (PEP 703) — the experimental builds where the GIL can be disabled. But even on standard Python, Bengal's architecture pays dividends: a provenance-based incremental system, a snapshot engine for lock-free parallel rendering, and a unified effect tracer that replaced 13 dependency detectors with one model.

---

## Run it with free-threaded Python

First, the practical bit. Bengal runs on standard Python and on free-threaded builds:

```bash
uv python install 3.14t
uv run --python=3.14t bengal build
```

Bengal detects free-threading at runtime and uses `ThreadPoolExecutor` when available. Same code paths either way — you just get real parallelism when the GIL is disabled.

---

## 1. Detecting free-threaded Python

Bengal doesn't assume free-threading — it detects it at runtime:

```python
def is_free_threaded() -> bool:
    if hasattr(sys, "_is_gil_enabled"):
        try:
            return not sys._is_gil_enabled()
        except (AttributeError, TypeError):
            pass
    try:
        import sysconfig
        return sysconfig.get_config_var("Py_GIL_DISABLED") == 1
    except (ImportError, AttributeError):
        pass
    return False
```

When `is_free_threaded()` returns `True`, Bengal spins up a `ThreadPoolExecutor` for page rendering. Each worker thread gets its own rendering pipeline (stored in `threading.local()`), so there's no contention on the hot path.

**Learning:** `sys._is_gil_enabled()` is the authoritative runtime check. The `sysconfig` fallback helps when the build was configured with `--disable-gil` but you're not sure at runtime.

---

## 2. Immutable snapshots for lock-free rendering

The trick to parallel rendering isn't just spawning threads. It's avoiding locks in the hot path.

Bengal uses a **snapshot engine**: after content discovery, the entire site is frozen into immutable dataclasses (`PageSnapshot`, `SectionSnapshot`, `SiteSnapshot`). All navigation trees, taxonomy indexes, and page metadata are pre-computed. During rendering, workers only read from these snapshots — no shared mutable state, no locks.

```python
@dataclass(frozen=True, slots=True)
class PageSnapshot:
    """Immutable page snapshot. Thread-safe by design."""
    title: str
    href: str
    source_path: Path
    parsed_html: str
    section: SectionSnapshot | None = None
    next_page: PageSnapshot | None = None
    prev_page: PageSnapshot | None = None
```

That design eliminated an entire tier of locks. Previously, `NavTreeCache` and `Renderer._cache_lock` were acquired during parallel rendering. Now, `SiteSnapshot.nav_trees` and `SiteSnapshot.top_level_*` are pre-computed in the snapshot builder. The rendering phase is lock-free.

---

## 3. Context propagation into worker threads

`ThreadPoolExecutor.submit()` does not inherit the calling thread's `ContextVar` values. If your template filters or logging use `ContextVar`s, they'll be empty in worker threads unless you propagate them.

Bengal uses `contextvars.copy_context().run` as the *callable* passed to `submit`:

```python
ctx = contextvars.copy_context()
future_to_page = {
    executor.submit(ctx.run, process_page_with_pipeline, page): page
    for page in batch
}
```

**Learning:** `executor.submit(ctx.run, fn, arg)` runs `fn(arg)` inside a copy of the parent's context. That copy includes all `ContextVar` values from the moment the main thread called `copy_context()`.

---

## 4. Thread-local vs ContextVar — when to use which

Bengal uses two concurrency primitives deliberately:

- **`threading.local()`** — for per-worker state: rendering pipelines, parsers. Each `ThreadPoolExecutor` worker gets its own. Correct for "one pipeline per thread."
- **`ContextVar`** — for per-task context: asset manifests, resolution stats. Propagates across `contextvars.copy_context().run()`. Correct for "context that should follow the task."

The pipeline itself lives in `thread_local`. The orchestrator wraps `executor.submit` with `ctx.run` so that ContextVars propagate into workers. Right primitive for the job.

**Learning:** Don't mix them. `ContextVar` is for "context that propagates with the task." `threading.local()` is for "one instance per worker thread."

---

## 5. Lock ordering when locks are unavoidable

Some state *does* need locks — caches, provenance stores, taxonomy indexes. Bengal documents a **global lock ordering** (Tier 0 → Tier 1 → … → Tier 7) that every developer must follow. Acquire locks in ascending tier order; never hold a higher-tier lock while acquiring a lower-tier one. That discipline prevents deadlocks under free-threading.

The concurrency module (`bengal/concurrency.py`) maintains a full inventory: which lock protects what, and in which tier. Verbose, but it makes the threading model auditable.

---

## 6. Provenance over manual dependencies

Incremental builds are where many SSGs get messy. "Which pages depend on this template? Which taxonomy keys does this page invalidate?" — usually answered by a patchwork of detectors.

Bengal uses **content-addressed provenance** instead. Each rendered output is hashed with everything that influenced it: source files, templates, cascade data, taxonomy. When a file changes, Bengal recomputes provenance and filters — only outputs whose provenance changed need rebuilding. No manual dependency graph. The previous system was ~30x slower; provenance made incremental builds practical for large sites.

---

## 7. Effect tracing — one model to replace 13 detectors

Even with provenance, you need to *record* what each render depends on and invalidates. Bengal's `EffectTracer` does that with a single unified model:

- **Effect.depends_on** — source file, layout, includes, parent `_index.md`, data files
- **Effect.invalidates** — taxonomy keys, generated pages

Thirteen detector classes were collapsed into one. During rendering, effects are recorded. When files change, `tracer.invalidated_by(changed_files)` returns the precise set of outputs to rebuild. Thread-safe, because it only reads from the frozen `SiteSnapshot`.

---

## What this means in practice

On free-threaded Python 3.14t with `PYTHON_GIL=0`, Bengal can render hundreds of pages in parallel without GIL contention. On standard Python, the same architecture runs; you just get sequential rendering until you switch interpreters.

The bigger win might be incremental builds. Provenance + effect tracing means rebuilds are fast and correct. Change one file, rebuild only what's affected — no stale cache, no over-invalidation.

Bengal is open source. Try it with a free-threaded build and see how it scales.

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Bengal documentation](https://lbliii.github.io/bengal/)
