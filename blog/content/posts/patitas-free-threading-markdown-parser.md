---
type: blog
title: "Patitas — A Markdown Parser Built for Parallel Parsing"
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
- python-3.14
- nogil
- commonmark
- redos
category: technology
description: "How Patitas achieves lock-free parallel Markdown parsing with immutable AST, ContextVar config, and an O(n) ReDoS-proof parser — 6.6x speedup on 8 cores"
params:
  author: kida
---

# Patitas — A Markdown Parser Built for Parallel Parsing

Most Markdown parsers have two problems that nobody talks about in the same conversation: **security** and **concurrency**.

The security problem is ReDoS. Regex-based lexers — and most Markdown parsers use regex — can backtrack catastrophically on malicious input. A carefully crafted string can lock your parser for minutes or hours. If your parser runs in a web app, that's a denial-of-service vector.

The concurrency problem is shared mutable state. Parser instances accumulate state: config dicts, cached ASTs, render context. Under free-threaded Python where the GIL no longer serializes access, threads race on that state.

Patitas solves both. It's a pure-Python Markdown parser for Python 3.14t — CommonMark 0.31.2 compliant, O(n) guaranteed (no regex in the hot path), and designed so that `executor.map(parse, docs)` just works.

---

:::{tip} Series context
This is **Part 3 of 6** in *Free-Threading in the Bengal Ecosystem*. Patitas is the Markdown layer — used by [Bengal](/blog/posts/bengal-free-threading-architecture/) for content parsing and by [Rosettes](/blog/posts/rosettes-free-threading-syntax-highlighter/) for code block highlighting.
:::

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

No special API. No locks. Just `executor.map(parse, docs)`.

---

## Performance

:::{list-table} Parallel parsing (1,000 documents, free-threaded 3.14t)
:header-rows: 1

- - Threads
  - Speedup
- - 1
  - 1.0x (baseline)
- - 2
  - ~2.0x
- - 4
  - ~3.8x
- - 8
  - ~6.6x
:::

Near-linear scaling on 8 cores. The architecture that makes this possible: immutable AST, ContextVar for config, single-use lexer/parser instances. No shared mutable state between parses.

---

## The ReDoS problem

:::{warning} Why this matters
Many Markdown parsers use regex patterns like `r'\[([^\]]*)\]\(([^)]*)\)'` for links. On pathological input — say, `[` repeated 50 times without a closing `]` — the regex engine backtracks exponentially. In a web application, an attacker can submit Markdown that locks your parser thread for seconds or minutes.
:::

Patitas uses a hand-written finite state machine instead:

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

---

## Immutable AST from the start

All AST nodes are frozen dataclasses with `tuple` children:

```python
@dataclass(frozen=True, slots=True)
class Heading(Node):
    level: int
    children: tuple[Inline, ...]

@dataclass(frozen=True, slots=True)
class Strong(Node):
    children: tuple[Inline, ...]
```

:::{tab-set}
:::{tab-item} Why tuple, not list
`tuple` is immutable — no one can append, remove, or reorder children after construction. Combined with `frozen=True`, this means AST nodes are hashable, comparable via `==`, and safe to pass between threads without copying.
:::
:::{tab-item} Why this enables parallelism
Immutable AST means safe to share across threads. No one can mutate a node. Structural diff, transforms, and incremental parsing all build on that guarantee. When multiple threads parse different documents, their ASTs can coexist without any synchronization.
:::
:::

---

## ContextVar for parse configuration

Parse config (tables, footnotes, math, directive registry) needs to be available deep in the parser. Storing it on a shared `Markdown` instance would require locks when multiple threads parse with different configs.

Patitas uses `ContextVar` instead:

```python
_parse_config: ContextVar[ParseConfig] = ContextVar(
    "parse_config", default=_DEFAULT_CONFIG
)
```

Each thread has its own config. Thread A can parse with tables enabled while thread B parses with math enabled — they don't interfere. No locks.

---

## Single-use lexer and parser

The lexer and parser hold mutable state during a parse — position, line number, block stack. Instead of making them thread-safe with locks, Patitas makes them **single-use**:

```python
# Each parse() call creates fresh instances
parser = Parser(source, source_file=source_file)
blocks = parser.parse()
```

Create, use, discard. No shared mutable state between parses. The `parse()` entry point handles config via ContextVar, creates the parser, runs it, and returns the immutable AST. Parallelism is trivial: `executor.map(parse, docs)`.

:::{tip} Pattern
Single-use instances are often simpler than lock-protected shared instances. The cost of creating a parser is small compared to the parse itself. The benefit is zero contention.
:::

---

## Incremental parsing with frozen trees

When the user edits one paragraph in a 100KB document, re-parsing everything is wasteful. Patitas has `parse_incremental()` — re-parse only the affected region and splice:

```python
def parse_incremental(
    new_source, previous, edit_start, edit_end, new_length, *,
    source_file=None,
) -> Document:
    # Find affected block range
    first_affected, last_affected = _find_affected_range(
        old_blocks, edit_start, edit_end
    )
    # Re-parse the affected region only
    new_blocks = _parse_region(region_source, reparse_start, source_file)
    # Splice: unchanged prefix + new middle + adjusted suffix
    result_children = (*before, *new_blocks, *adjusted_after)
    return Document(location=loc, children=result_children)
```

The existing AST is immutable — you can slice it, pass parts to other threads, and build a new Document from pieces. `dataclasses.replace()` adjusts offsets in the "after" blocks. ~200x faster than full re-parse for small edits.

---

## Structural diff on frozen nodes

Patitas has `diff_documents(old, new)` — structural diff for ASTs. The fast path: `==` on frozen nodes. If two nodes are equal, skip the entire subtree:

```python
def diff_documents(
    old: Document, new: Document, *, recursive: bool = False
) -> tuple[ASTChange, ...]:
    """Unchanged subtrees are skipped in O(1) via == on frozen nodes."""
```

Frozen dataclasses are hashable and comparable. Structural equality is O(n) in the worst case, but when most of the tree is unchanged (the typical incremental edit), the fast path dominates.

---

## Immutable transforms

`transform(doc, fn)` walks the AST bottom-up and produces a new tree:

```python
def transform(doc: Document, fn: Callable[[Node], Node | None]) -> Document:
    """Apply a function to every node, returning a new tree.
    The original tree is untouched.
    """
```

Pure function. No side effects. Safe to call from any thread. Sanitization policies (`strip_html`, `strip_dangerous_urls`) use `transform()` for AST rewriting — composable, immutable, thread-safe.

---

## What this means in practice

On free-threaded Python 3.14t, `executor.map(parse, docs)` scales to ~6.6x on 8 cores. 1,000 documents parsed in parallel with no locks and no special API. The O(n) lexer means you can safely parse untrusted Markdown in web applications without ReDoS risk.

Patitas is the Markdown layer in the Bengal ecosystem — it feeds parsed content to [Bengal](/blog/posts/bengal-free-threading-architecture/) for rendering and uses [Rosettes](/blog/posts/rosettes-free-threading-syntax-highlighter/) for syntax highlighting in code blocks.

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Patitas documentation](https://lbliii.github.io/patitas/)
- [Patitas source](https://github.com/lbliii/patitas)
- **Next in series:** [Rosettes — Immutable State Machines for Thread-Safe Syntax Highlighting](/blog/posts/rosettes-free-threading-syntax-highlighter/)
