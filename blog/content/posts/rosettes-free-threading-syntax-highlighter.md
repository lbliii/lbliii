---
type: blog
title: Rosettes — Immutable State Machines for Thread-Safe Syntax Highlighting
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
category: technology
description: How Rosettes achieves lock-free parallel highlighting with local variables only, frozen lookup tables, and O(n) ReDoS-proof lexers — techniques for the free-threading ecosystem
params:
  author: kida
---

# Rosettes — Immutable State Machines for Thread-Safe Syntax Highlighting

Syntax highlighters have two common problems: regex-based lexers that are vulnerable to ReDoS, and shared mutable state that causes races when multiple threads highlight code. Under free-threaded Python, that second problem becomes visible — threads can overwrite each other's lexer state, config, or token buffers.

Rosettes is a pure-Python syntax highlighter for Python 3.14t. Hand-written state machines, O(n) guaranteed, zero ReDoS risk.

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
] * 20  # 60 blocks

results = highlight_many(blocks)
print(f'Highlighted {len(results)} blocks')
"
```

For 8+ blocks, `highlight_many()` uses `ThreadPoolExecutor`. On Python 3.14t with free-threading, that gives real parallelism — 1.5–2x speedup for 50+ blocks. No locks. Each lexer uses only local variables.

---

## 1. Local variables only — no instance state

The critical rule: **no mutable instance state during tokenization.** Lexer state lives in local variables:

```python
# ❌ WRONG: Storing state in instance variables
self.current_line = 1  # NOT thread-safe!

# ✅ CORRECT: Use local variables
line = 1
```

The Python lexer main loop:

```python
def tokenize(self, code, start=0, end=None):
    pos = start
    length = end if end is not None else len(code)
    line = 1
    line_start = start

    while pos < length:
        char = code[pos]
        col = pos - line_start + 1

        if char in self.WHITESPACE:
            start = pos
            start_line = line
            while pos < length and code[pos] in self.WHITESPACE:
                if code[pos] == "\n":
                    line += 1
                    line_start = pos + 1
                pos += 1
            yield Token(TokenType.WHITESPACE, code[start:pos], start_line, col)
            continue
        # ...
```

`pos`, `line`, `line_start` — all local. No `self.state`. Multiple threads can call `tokenize()` on the same lexer instance concurrently. No locks needed.

**Learning:** If your hot path uses only locals, you get thread-safety for free. The lexer instance is effectively stateless during tokenization.

---

## 2. Frozen lookup tables

Keyword and character-set lookups use `frozenset` — O(1) membership and immutable:

```python
# Keyword and builtin lookup tables (frozen for thread safety)
_KEYWORDS: frozenset[str] = frozenset(
    {"def", "class", "if", "else", "return", "match", "case", "type", ...}
)

# Base class character sets (frozen for thread safety)
DIGITS: frozenset[str] = frozenset("0123456789")
IDENT_START: frozenset[str] = frozenset("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_")
WHITESPACE: frozenset[str] = frozenset(" \t\n\r\f\v")
```

In the tokenize loop: `if word in _KEYWORDS`, `if char in self.WHITESPACE`. No one can mutate these. Safe to share across threads.

**Learning:** Use `frozenset` for read-only lookup tables. O(1) membership, immutable, thread-safe.

---

## 3. No regex — scan_while instead

Regex can backtrack catastrophically (ReDoS). Rosettes uses helper functions that advance position without backtracking:

```python
# ❌ WRONG: Using regex for matching
match = re.match(r'\d+', code[pos:])  # ReDoS vulnerable!

# ✅ CORRECT: Use scan_while helper
end_pos = scan_while(code, pos, self.DIGITS)
```

```python
def scan_while(code: str, pos: int, char_set: frozenset[str]) -> int:
    """Advance position while characters are in char_set."""
    length = len(code)
    while pos < length and code[pos] in char_set:
        pos += 1
    return pos
```

Single pass. No backtracking. O(n) guaranteed. Tests run pathological inputs (nested parens, repeated escapes) with a 1s timeout and verify linear scaling — doubling input size should roughly double time, not explode.

**Learning:** Hand-written scanners beat regex for security and predictability. `scan_while` / `scan_until` are the building blocks.

---

## 4. Immutable tokens

Tokens are `NamedTuple` — immutable, minimal memory, safe to share:

```python
class Token(NamedTuple):
    """Immutable token — thread-safe, minimal memory."""
    type: TokenType
    value: str
    line: int = 1
    column: int = 1
```

No defensive copying when passing tokens between workers. The formatter receives tokens, formats them, returns a string. No shared mutable state.

---

## 5. Frozen config and formatters

Lexer config, highlight config, and formatters are frozen dataclasses:

```python
@dataclass(frozen=True, slots=True)
class LexerConfig:
    ...

@dataclass(frozen=True, slots=True)
class HighlightConfig:
    hl_lines: frozenset[int] = frozenset()
    ...

@dataclass(frozen=True, slots=True)
class HtmlFormatter:
    """Thread-safe: all state is immutable or local to method calls."""
```

Instances can be shared across threads. No one mutates them. Formatting uses only local state (e.g. a StringBuilder).

---

## 6. highlight_many() — parallel by default

For 8+ blocks, Rosettes uses `ThreadPoolExecutor`:

```python
# For small batches, sequential is faster (thread overhead)
if len(items_list) < 8:
    return [_highlight_one(it) for it in items_list]

# Optimal worker count: 4 workers is sweet spot
max_workers = min(4, os.cpu_count() or 4)
with ThreadPoolExecutor(max_workers=max_workers) as executor:
    return list(executor.map(_highlight_one, items_list))
```

Each worker calls `highlight()` (or `tokenize()` + `format()`). Lexers use locals only. Formatters are frozen. No contention. On 3.14t, real parallelism.

**Learning:** Batch threshold matters. For small batches, thread overhead dominates. Rosettes uses 8 as the cutoff; benchmarking showed 4 workers as optimal.

---

## 7. Declaring GIL independence (PEP 703)

Rosettes explicitly declares that it doesn't rely on the GIL:

```python
def __getattr__(name: str) -> object:
    if name == "_Py_mod_gil":
        # Signal: this module is safe for free-threading
        # 0 = Py_MOD_GIL_NOT_USED
        return 0
    raise AttributeError(...)
```

`_Py_mod_gil = 0` tells the runtime and ecosystem that this module is designed for free-threading.

---

## What this means in practice

On free-threaded Python 3.14t, `highlight_many()` scales. 100 code blocks, 4 workers — ~2.5x speedup. No locks, no special API. The architecture — local variables only, frozen lookup tables, immutable tokens and config, no regex — makes parallelism drop-in.

Rosettes powers syntax highlighting in Patitas (Markdown) and Bengal. It's the highlighting layer in the Bengal ecosystem — a stack built for Python's free-threaded future.

---

## Further reading

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Rosettes documentation](https://lbliii.github.io/rosettes/)
