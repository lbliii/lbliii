---
type: blog
title: "Rosettes — Immutable State Machines for Thread-Safe Syntax Highlighting"
date: '2026-03-02'
series:
  id: nogil
  name: "Free-Threading in the Bengal Ecosystem"
  part: 4
  total: 6
tags:
- rosettes
- python
- free-threading
- syntax-highlighting
- lexer
- python-3.14
- nogil
- pygments-alternative
category: technology
description: "How Rosettes achieves lock-free parallel syntax highlighting with local-only state, frozen lookup tables, and O(n) hand-written lexers — 2.5x speedup on 4 cores"
params:
  author: kida
---

# Rosettes — Lock-Free Syntax Highlighting for Free-Threaded Python

Rosettes is the simplest library in the Bengal stack, and that's the point. A syntax highlighter has one job: turn source code into colored tokens. The design space is small enough that you can get the threading model exactly right.

The rule is straightforward: **no mutable instance state during tokenization.** All lexer state lives in local variables. If your hot path uses only locals, you get thread-safety for free — no locks, no `ContextVar`, no copy-on-write. Just functions that take input and produce output.

---

:::{tip} Series context
This is **Part 4 of 6** in *Free-Threading in the Bengal Ecosystem*. Rosettes is the syntax highlighting layer — used by [Patitas](/blog/posts/patitas-free-threading-markdown-parser/) for code blocks and [Bengal](/blog/posts/bengal-free-threading-architecture/) for build-time highlighting.
:::

---

## Run it with free-threaded Python

```bash
uv python install 3.14t
uv run --python=3.14t python -c "
from rosettes import highlight_many

blocks = [
    ('def foo(): pass', 'python'),
    ('const x = 1;', 'javascript'),
    ('fn main() {}', 'rust'),
] * 20

results = highlight_many(blocks)
print(f'Highlighted {len(results)} blocks')
"
```

For 8+ blocks, `highlight_many()` uses `ThreadPoolExecutor`. On Python 3.14t, that's real parallelism.

---

## Performance

:::{list-table} Parallel highlighting (100 blocks, free-threaded 3.14t)
:header-rows: 1

- - Workers
  - Time
  - Speedup
- - 1 (sequential)
  - 0.04 s
  - 1.0x
- - 2
  - 0.02 s
  - 1.6x
- - 4
  - 0.02 s
  - 2.5x
:::

The batch threshold matters. For small batches, thread overhead dominates. Rosettes uses 8 blocks as the cutoff — below that, sequential is faster. Benchmarking showed 4 workers as the sweet spot for throughput.

---

## Local variables only — no instance state

This is the core pattern. Lexer state lives in local variables, never in `self`:

:::{tab-set}
:::{tab-item} Wrong — instance state
```python
def tokenize(self, code):
    self.pos = 0          # Shared across threads!
    self.line = 1         # Race condition
    self.line_start = 0   # Data corruption
    while self.pos < len(code):
        ...
```
:::
:::{tab-item} Right — local variables
```python
def tokenize(self, code, start=0, end=None):
    pos = start               # Local to this call
    length = end or len(code)
    line = 1
    line_start = start

    while pos < length:
        char = code[pos]
        col = pos - line_start + 1
        ...
```
:::
:::

`pos`, `line`, `line_start` — all local. Multiple threads can call `tokenize()` on the same lexer instance concurrently. The lexer instance is effectively stateless during tokenization.

---

## Frozen lookup tables

Keyword and character-set lookups use `frozenset` — O(1) membership and immutable:

```python
_KEYWORDS: frozenset[str] = frozenset(
    {"def", "class", "if", "else", "return", "match", "case", "type", ...}
)

DIGITS: frozenset[str] = frozenset("0123456789")
IDENT_START: frozenset[str] = frozenset(
    "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_"
)
WHITESPACE: frozenset[str] = frozenset(" \t\n\r\f\v")
```

In the tokenize loop: `if word in _KEYWORDS`, `if char in self.WHITESPACE`. Nobody can mutate these. Safe to share across threads without any protection.

---

## No regex — scan_while instead

Like [Patitas](/blog/posts/patitas-free-threading-markdown-parser/), Rosettes avoids regex in the hot path. The building block is `scan_while`:

```python
def scan_while(code: str, pos: int, char_set: frozenset[str]) -> int:
    """Advance position while characters are in char_set."""
    length = len(code)
    while pos < length and code[pos] in char_set:
        pos += 1
    return pos
```

Single pass. No backtracking. O(n) guaranteed. Tests run pathological inputs (nested parens, repeated escapes) with a 1-second timeout to verify linear scaling.

:::{warning} ReDoS in syntax highlighters
Syntax highlighters are often overlooked as a ReDoS vector. If your highlighter runs server-side on user-submitted code (documentation sites, paste services, code review tools), a regex-based lexer is an attack surface. Rosettes eliminates that by construction.
:::

---

## Immutable everything else

Tokens, config, and formatters are all immutable:

```python
class Token(NamedTuple):
    type: TokenType
    value: str
    line: int = 1
    column: int = 1

@dataclass(frozen=True, slots=True)
class HighlightConfig:
    hl_lines: frozenset[int] = frozenset()
    ...

@dataclass(frozen=True, slots=True)
class HtmlFormatter:
    config: FormatConfig = field(default_factory=FormatConfig)
    ...
```

No defensive copying when passing tokens between workers. The formatter receives tokens, formats them, returns a string. No shared mutable state at any layer.

---

## highlight_many() — parallel by default

```python
if len(items_list) < 8:
    return [_highlight_one(it) for it in items_list]

max_workers = min(4, os.cpu_count() or 4)
with ThreadPoolExecutor(max_workers=max_workers) as executor:
    return list(executor.map(_highlight_one, items_list))
```

Each worker calls `highlight()`. Lexers use locals only. Formatters are frozen. On 3.14t, real parallelism. On GIL builds, it falls back gracefully — the code is correct either way, you just don't get the speedup.

---

## What this means in practice

Rosettes is small — ~55 language lexers, pure Python, zero dependencies. The threading model is the simplest in the stack: use local variables for mutable state, `frozenset` for lookup tables, frozen dataclasses for config and output. No locks, no `ContextVar`, no copy-on-write. When the hot path is already stateless, thread-safety is free.

Rosettes powers syntax highlighting in [Patitas](/blog/posts/patitas-free-threading-markdown-parser/) (Markdown code blocks) and [Bengal](/blog/posts/bengal-free-threading-architecture/) (build-time highlighting for documentation sites).

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Rosettes documentation](https://lbliii.github.io/rosettes/)
- [Rosettes source](https://github.com/lbliii/rosettes)
- **Next in series:** [Pounce — Thread-Based ASGI Workers](/blog/posts/pounce-free-threading-asgi-server/)
