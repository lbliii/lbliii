---
type: blog
title: Patitas — A Markdown Parser Built for Parallel Parsing
date: '2026-03-02'
series:
  id: nogil
  name: "Free-Threading in the Bengal Ecosystem"
  part: 3
  total: 6
tags:
- patitas
- python
- free-threading
- markdown
- parsing
category: technology
description: How Patitas achieves lock-free parallel parsing with immutable AST, ContextVar config, and an O(n) ReDoS-proof lexer — techniques for the free-threading ecosystem
params:
  author: kida
---

# Patitas — A Markdown Parser Built for Parallel Parsing

Markdown parsers have two common problems: regex-based lexers that are vulnerable to ReDoS, and shared mutable state that makes parallel parsing unsafe. Under free-threaded Python, that second problem becomes visible — threads can race on config dicts, parser state, or cached ASTs.

Patitas is a pure-Python Markdown parser for Python 3.14+ that parses to a typed AST and renders to HTML. It's CommonMark 0.31.2 compliant, ReDoS-proof, and designed for free-threading.

---

## Run it with free-threaded Python

```bash
uv python install 3.14t
uv run --python=3.14t python -c "
from concurrent.futures import ThreadPoolExecutor
from patitas import parse

docs = ['# Doc ' + str(i) for i in range(1000)]
with ThreadPoolExecutor(max_workers=8) as ex:
    results = list(ex.map(parse, docs))
print(f'Parsed {len(results)} documents in parallel')
"
```

No special API. No locks. Just `executor.map(parse, docs)`. On 8 cores with free-threading, Patitas scales near-linearly — ~6.6x speedup. Same code on standard Python runs; you just don't get parallelism.

---

## 1. Immutable AST from the start

All AST nodes are frozen dataclasses with `tuple` children:

```python
@dataclass(frozen=True, slots=True)
class Node:
    """Base class for all AST nodes.
    Nodes are immutable for thread-safety.
    """
    location: SourceLocation

@dataclass(frozen=True, slots=True)
class Heading(Node):
    level: int
    children: tuple[Inline, ...]

@dataclass(frozen=True, slots=True)
class Strong(Node):
    children: tuple[Inline, ...]
```

Immutable AST means safe to share across threads. No one can mutate a node. Structural diff, transforms, and incremental parsing all build on that guarantee.

**Learning:** Use `tuple` for children, not `list`. Frozen + tuple = hashable, comparable via `==`, and safe to pass between threads without copying.

---

## 2. ContextVar for parse configuration

Parse config (tables, footnotes, math, directive registry) needs to be available to the parser. Storing it on a shared `Markdown` instance would require locks when multiple threads parse with different configs.

Patitas uses `ContextVar` instead:

```python
_parse_config: ContextVar[ParseConfig] = ContextVar(
    "parse_config",
    default=_DEFAULT_CONFIG,
)

def get_parse_config() -> ParseConfig:
    return _parse_config.get()

def set_parse_config(config: ParseConfig) -> None:
    _parse_config.set(config)
```

Each thread has its own config. The parser reads via `get_parse_config()`. No locks. Thread A can parse with tables enabled while thread B parses with math enabled — they don't interfere.

**Learning:** ContextVar is ideal for "config that varies per logical task." Per-thread, per-request, per-async-task — all work without shared mutable state.

---

## 3. ContextVar for per-render state

The HTML renderer needs to collect headings during render (for TOC). The old pattern: store `_last_context` on the renderer instance. Under concurrency, that's a race — thread B overwrites thread A's context before A calls `get_headings()`.

Patitas uses a ContextVar:

```python
_render_context: ContextVar[RenderContext | None] = ContextVar(
    "_render_context", default=None
)

def render(self, node: Document) -> str:
    ctx = RenderContext()  # Fresh per call
    # ... render, populating ctx.headings ...
    _render_context.set(ctx)  # For get_headings()
    return sb.build()

def get_headings(self) -> list[HeadingInfo]:
    ctx = _render_context.get()
    return ctx.headings.copy() if ctx else []
```

Each render creates its own `RenderContext`. `get_headings()` returns data from the current thread's context. Multiple threads can share one `HtmlRenderer` instance safely.

**Learning:** When you need "state from the most recent X in this context," ContextVar beats instance attributes. No cross-thread pollution.

---

## 4. Single-use lexer and parser

The lexer and parser hold mutable state during a parse. Instead of making them thread-safe with locks, Patitas makes them single-use:

```python
# Lexer: "Create one per source string. All state is instance-local."
# Parser: "Parser instances are single-use. Create one per parse operation."
```

Each `parse()` call creates a fresh lexer and parser. No shared mutable state between parses. The `parse()` entry point handles config via ContextVar, creates the parser, runs it, and returns the immutable AST. Parallelism is trivial: `executor.map(parse, docs)`.

**Learning:** Single-use instances are often simpler than lock-protected shared instances. Create, use, discard. No contention.

---

## 5. O(n) lexer — no ReDoS

Many Markdown parsers use regex. Regex can backtrack catastrophically on malicious input (ReDoS). Patitas uses a hand-written finite state machine:

```python
"""State-machine lexer with O(n) guaranteed performance.

Uses a window-based approach:
1. Scan to end of line (find window)
2. Classify the line (pure logic, no position changes)
3. Commit position (always advances)

No regex in the hot path. Zero ReDoS vulnerability by construction.
"""
```

Single-character lookahead. No backtracking. Processing time scales linearly with input length. Safe for untrusted input in web apps and APIs.

**Learning:** For parsers that handle user input, O(n) guarantees matter. FSM lexers are more work upfront but eliminate a whole class of security issues.

---

## 6. Incremental parsing with frozen trees

When the user edits one paragraph in a 100KB document, re-parsing everything is wasteful. Patitas has `parse_incremental(new_source, prev_doc, edit_start, edit_end)` — re-parse only the affected region and splice:

```python
# Find affected block range
first_affected, last_affected = _find_affected_range(old_blocks, edit_start, edit_end)

# Re-parse the affected region only
region_source = new_source[reparse_start:reparse_end_new]
new_blocks = _parse_region(region_source, reparse_start, source_file)

# Splice: before + new + after (with adjusted offsets)
before = old_blocks[:first_affected]
adjusted_after = _adjust_offsets(after_blocks, offset_delta)
result_children = (*before, *new_blocks, *adjusted_after)
return Document(location=loc, children=result_children)
```

The existing AST is immutable — we can slice it, pass parts to other threads, and build a new Document from pieces. `dataclasses.replace()` updates offsets in the "after" blocks. ~200x faster than full re-parse for small edits.

**Learning:** Immutable trees enable splicing. You can't splice in-place into a frozen structure, but you can build a new one from unchanged prefix + new middle + adjusted suffix.

---

## 7. Structural diff on frozen nodes

Patitas has `diff_documents(old, new)` — structural diff for ASTs. The fast path: `==` on frozen nodes. If two nodes are equal, skip the entire subtree:

```python
"""Unchanged subtrees are skipped in O(1) via == on frozen nodes.

Algorithm:
1. Walk both trees in parallel by child index.
2. For each position, compare nodes via ==.
3. If equal, skip the subtree (fast path).
4. If different, emit added/removed/modified.
"""
```

Frozen dataclasses are hashable and comparable. Structural equality is O(n) in the worst case, but when most of the tree is unchanged, the fast path dominates.

**Learning:** Immutability enables cheap equality. Mutable nodes would need custom comparison; frozen nodes get it for free.

---

## 8. Immutable transform

`transform(doc, fn)` walks the AST bottom-up and produces a new tree:

```python
def transform(doc: Document, fn: Callable[[Node], Node | None]) -> Document:
    """Apply a function to every node, returning a new tree.

    Since all nodes are frozen dataclasses, this produces a new immutable tree.
    The original tree is untouched.
    """
```

Pure function. No side effects. Safe to call from any thread. Sanitization policies (`strip_html | strip_dangerous_urls`) use `transform()` for AST rewriting — composable, immutable, thread-safe.

---

## What this means in practice

On free-threaded Python 3.14t, `executor.map(parse, docs)` scales. 1,000 documents, 8 threads — ~6.6x speedup. No locks, no special API. The architecture — immutable AST, ContextVar for config and render state, single-use lexer/parser, O(n) lexer — makes parallelism drop-in.

Patitas is the Markdown layer in the Bengal ecosystem (Bengal, Kida, Chirp, Pounce, Rosettes) — a stack built for Python's free-threaded future.

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Patitas documentation](https://lbliii.github.io/patitas/)
