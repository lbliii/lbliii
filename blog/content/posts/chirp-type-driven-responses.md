---
type: blog
title: "Type-Driven Responses — How Chirp Eliminates make_response()"
date: '2026-03-06'
tags:
- chirp
- python
- htmx
- web-framework
- content-negotiation
- hypermedia
- architecture
- design-patterns
category: technology
description: "In Chirp, the return type is the intent. Template, Fragment, Page, Stream, Suspense, EventStream — each type maps to exactly one response shape. No conditionals. No make_response(). No forgetting to check headers."
params:
  author: kida
---

# Type-Driven Responses — How Chirp Eliminates make_response()

Every Python web framework has the same problem: the route handler knows what it wants to do, but the framework doesn't. Flask returns `make_response()`. Django returns `HttpResponse()`. FastAPI returns dicts or `JSONResponse()`. The handler does the work of deciding *what* to return and *how* to return it.

Chirp inverts this. The handler returns a **type** — `Template`, `Fragment`, `Page`, `Stream`, `Suspense`, `EventStream` — and the framework's content negotiator decides the response shape. The return type *is* the intent.

---

## The negotiation model

When a route handler returns, Chirp's `negotiate()` function inspects the value using structural pattern matching:

```python
match value:
    case Response():      # pass through
    case Redirect():      # 302 + Location
    case FormAction():    # fragments or redirect
    case Template():      # full page render
    case Fragment():      # block render
    case Page():          # full or fragment based on request
    case Action():        # empty body + HX headers
    case ValidationError(): # 422 + fragment
    case OOB():           # primary + hx-swap-oob
    case Stream():        # progressive HTML
    case Suspense():      # shell + deferred OOB
    case EventStream():   # SSE
    case str():           # text/html
    case bytes():         # octet-stream
    case dict() | list(): # JSON
```

No if-chains. No isinstance ladders. Python 3.10's `match` gives you exhaustive, readable dispatch. Each case maps to exactly one response shape.

---

## Why this matters for htmx

The htmx pattern has a fundamental tension: the same route often needs to return a full page *or* a fragment, depending on whether the request came from a browser navigation or an htmx swap.

In Flask, you write this:

```python
@app.route("/search")
def search():
    books = find_books(request.args.get("q", ""))
    if request.headers.get("HX-Request"):
        return render_template("search.html", books=books, block="results")
    return render_template("search.html", books=books)
```

Every route. Every time. Forget the check once and your htmx swap renders the full page inside a `div`.

In Chirp, you write this:

```python
@app.route("/search")
def search(request: Request):
    books = find_books(request.query.get("q", ""))
    return Page("search.html", "results", books=books, query=request.query.get("q"))
```

`Page` encapsulates the decision. If the request has `HX-Request: true` and isn't a history restore, the negotiator renders just the `results` block. Otherwise, it renders the full template. Same data. Same template. The type carries the intent.

---

## The return types

Each type encodes a specific rendering strategy:

:::{list-table} Chirp return types
:header-rows: 1

* - Type
  - Intent
  - Response
* - `Template(name, **ctx)`
  - Full page
  - Rendered HTML, 200
* - `Fragment(name, block, **ctx)`
  - Named block
  - Block HTML, 200
* - `Page(name, block, **ctx)`
  - Full or fragment
  - Depends on `request.is_fragment`
* - `Stream(name, **ctx)`
  - Progressive render
  - Chunked HTML, awaitables resolved concurrently
* - `Suspense(name, shell, blocks)`
  - Shell + deferred blocks
  - Instant first paint, OOB blocks fill in
* - `EventStream(generator)`
  - Server-sent events
  - SSE response
* - `OOB(main, *fragments)`
  - Primary + out-of-band swaps
  - Main HTML + `hx-swap-oob` fragments
* - `ValidationError(name, block, **ctx)`
  - Form error
  - 422 + fragment + optional `HX-Retarget`
* - `FormAction(redirect, fragments?)`
  - Post-submit
  - Fragments (htmx) or redirect (browser)
* - `Action(trigger?, refresh?)`
  - Side-effect only
  - Empty body + HX headers
* - `str`
  - Raw HTML
  - 200, text/html
* - `dict` / `list`
  - JSON API
  - 200, application/json
:::

---

## Real examples

### Fragment for htmx CRUD

```python
@app.route("/tasks", methods=["POST"])
def create_task(request: Request):
    task = store.add(request.form["title"])
    return Fragment("tasks.html", "task_item", task=task)
```

The handler doesn't think about response objects, content types, or status codes. It returns data shaped as intent.

### OOB for multi-region updates

```python
@app.route("/tasks/{id}/complete", methods=["POST"])
def complete_task(id: int):
    task = store.complete(id)
    return OOB(
        Fragment("tasks.html", "task_item", task=task),
        Fragment("tasks.html", "stats", stats=store.stats()),
    )
```

One return. Two regions update. The negotiator handles `hx-swap-oob` wrapping.

### Suspense for instant first paint

```python
@app.route("/dashboard")
async def dashboard():
    return Suspense(
        "dashboard.html",
        shell="shell",
        blocks={
            "metrics": fetch_metrics(),
            "activity": fetch_activity(),
        },
    )
```

The shell renders immediately. Deferred blocks resolve concurrently and stream in as OOB swaps. The user sees content instantly; heavy data fills in without a loading spinner.

### ValidationError for form errors

```python
@app.route("/signup", methods=["POST"])
def signup(request: Request):
    errors = validate(request.form, rules)
    if errors:
        return ValidationError("signup.html", "form", errors=errors)
    # ... create user
```

Returns 422, renders the form fragment with errors, and optionally retargets the swap. The browser-side htmx handles the rest.

### EventStream for live updates

```python
@app.route("/feed")
async def live_feed():
    async def events():
        async for event in bus.subscribe():
            yield SSEEvent(data=event.html, event="update")
    return EventStream(events())
```

The negotiator wraps it in an SSE response. No manual event formatting.

---

## FormAction — the POST/redirect/fragment split

Traditional web forms use Post-Redirect-Get. htmx forms want fragments back. `FormAction` handles both in one return:

```python
@app.route("/tasks", methods=["POST"])
def create_task(request: Request):
    task = store.add(request.form["title"])
    return FormAction(
        redirect="/tasks",
        fragments=[Fragment("tasks.html", "task_list", tasks=store.all())],
    )
```

- **htmx request** → renders the fragments, returns HTML
- **Browser POST** → 303 redirect to `/tasks`

One handler. Both clients work correctly. The negotiator reads `request.is_fragment` and dispatches.

---

## What the negotiator doesn't do

It doesn't guess. Every return type maps to exactly one response strategy. There's no content-type sniffing, no Accept header parsing, no fallback chains. If you return `Fragment`, you get a fragment. If you return `Template`, you get a full page. If you return `dict`, you get JSON.

This is deliberate. Implicit negotiation — like Rails' `respond_to` — creates behavior that's hard to predict and harder to debug. Chirp's negotiation is explicit: the type *is* the contract.

:::{tip} Pattern
Type-driven dispatch replaces conditionals with structure. Instead of `if request.is_fragment ... else ...` in every handler, encode the intent once in the return type and let the framework dispatch. The handler stays focused on business logic.
:::

---

## The match statement as architecture

Python's `match` statement (3.10+) makes this pattern natural. The negotiator is ~250 lines of code with no inheritance, no visitor pattern, no registry. Just a function that matches on types and produces responses.

Adding a new return type means:

1. Define the dataclass in `chirp.templating.returns`
2. Add a `case` branch in `negotiate()`
3. Contract checks validate that routes return supported types

That's it. No adapter interfaces. No plugin system. The simplicity is the point.

---

## Further reading

- [Chirp documentation](https://lbliii.github.io/chirp/)
- [Chirp source — negotiation.py](https://github.com/lbliii/chirp/blob/main/src/chirp/server/negotiation.py)
- [Part 6: Chirp — Free-Threaded Web Framework](/blog/posts/chirp-free-threading-web-framework/)
- [The Vertical Stack Thesis](/blog/posts/the-vertical-stack-thesis/)
