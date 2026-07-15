# Python Lesson 04 — Functions, Scope, Closures & Decorators

| | |
|---|---|
| **Track** | Python |
| **Lesson** | 04 of 15 |
| **Topic** | Function objects, argument binding, scoping rules, closures, and decorators |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–03 (especially the names-vs-objects model from Lesson 02) |

### Learning objectives
1. Explain what a function object actually *is* at runtime — code object, `__defaults__`, `__closure__`, `__globals__` — and why that explains most "weird" behaviour.
2. Bind arguments correctly using positional, keyword, `*args`, `**kwargs`, positional-only (`/`) and keyword-only (`*`) markers.
3. State the LEGB rule precisely and know when `global`/`nonlocal` are required (and when their presence is a design smell).
4. Build closures deliberately, and explain late binding well enough to debug it in someone else's code.
5. Write decorators — plain, parameterised, class-based, and stacked — that preserve introspection and don't corrupt signatures.
6. Reason about `functools.wraps`, `functools.lru_cache`, and `functools.partial` at the level of "what does this cost me at scale?"

---

## 1. Why this matters

Functions are the unit of reuse, and in Python they are also *objects you can inspect, store, wrap, and generate at runtime*. That second fact is what makes the entire modern Python ecosystem work. Every framework you will touch in your target roles is built on the material in this lesson:

- **FastAPI** reads your function's signature and type annotations to build validation, dependency injection, and OpenAPI docs. `@app.get("/items")` is a parameterised decorator that registers a closure.
- **Airflow / Dagster / Prefect** turn decorated functions into DAG tasks. `@task` is the whole authoring interface.
- **pytest** discovers plain functions and injects fixtures by matching parameter *names* to fixture functions.
- **Retry, caching, timing, auth, rate limiting, tracing** — the entire cross-cutting-concerns layer of a production service — is decorators.

At interview level, decorators are the classic separator between "writes Python" and "understands Python." A mid-level candidate can *use* `@lru_cache`. A senior candidate can tell you that it holds strong references to arguments and return values, that it makes the cached function un-garbage-collectable along with everything it ever returned, that it requires hashable arguments, and that on a method it keys on `self` and therefore leaks instances.

---

## 2. Theory

### 2.1 A function is an object

`def` is not a declaration. It is a **statement that executes at runtime**: it compiles a code object, builds a function object around it, and binds a name in the enclosing namespace. Everything you can do to an object you can do to a function.

```python
def greet(name: str, punct: str = "!") -> str:
    """Say hello."""
    return f"Hello, {name}{punct}"

greet.__name__        # 'greet'
greet.__doc__         # 'Say hello.'
greet.__defaults__    # ('!',)          <- default VALUES, evaluated once at def time
greet.__annotations__ # {'name': <class 'str'>, 'punct': <class 'str'>, 'return': <class 'str'>}
greet.__code__.co_varnames   # ('name', 'punct')
greet.__code__.co_argcount   # 2
greet.__module__      # '__main__'

greet.author = "you"  # functions have a __dict__; you can hang attributes on them
```

Three consequences that matter:

1. **Functions are first-class**: pass them, return them, store them in dicts. A dict of `{event_name: handler_function}` replaces most `if/elif` dispatch chains.
2. **Defaults are evaluated once**, at definition time, and live in `__defaults__` forever. This is the single most famous Python gotcha (§9).
3. **Decorators are just function application.** `@d` above `def f` means `f = d(f)`. There is no magic; there is only sugar.

### 2.2 Parameters vs arguments, and the full binding grammar

A **parameter** is the name in the `def`. An **argument** is the value at the call site. Python's binding grammar is richer than most languages':

```python
def f(pos_only, /, standard, *args, kw_only, **kwargs):
    ...
```

| Section | Meaning |
|---|---|
| before `/` | **positional-only** — cannot be passed by name (3.8+) |
| between `/` and `*` | standard — positional *or* keyword |
| `*args` | collects surplus positional arguments into a tuple |
| after `*` or `*args` | **keyword-only** — must be passed by name |
| `**kwargs` | collects surplus keyword arguments into a dict |

```python
def connect(host, port=5432, /, *, timeout=30, retries=3, **driver_opts):
    ...

connect("db.internal", 5432, timeout=5, sslmode="require")   # OK
connect(host="db.internal")                                  # TypeError: positional-only
```

**Why positional-only matters:** it frees you to rename parameters later without breaking callers, and it lets `**kwargs` legitimately contain a key called `host`. This is why the stdlib uses it heavily (`len(obj, /)`, `dict.get(key, default, /)`).

**Why keyword-only matters:** it kills the boolean-trap call site. `create_user("ann", True, False)` is unreadable and unreviewable; `create_user("ann", *, admin=True, verified=False)` forces callers into clarity. **Make every boolean and every tuning knob keyword-only.** This is the highest-value, lowest-cost API habit in this lesson.

Unpacking at the call site mirrors this:

```python
args = ("db.internal", 5432)
opts = {"timeout": 5, "retries": 1}
connect(*args, **opts)
```

### 2.3 Scope: LEGB, precisely

Name resolution searches four scopes, in order:

- **L**ocal — the current function's namespace.
- **E**nclosing — the local scopes of lexically enclosing functions, innermost first.
- **G**lobal — the module's namespace (`__main__`, or whatever module the function was *defined* in — not called from).
- **B**uiltins — the `builtins` module.

The critical, non-obvious rule: **assignment anywhere in a function body makes the name local for the entire body**, determined at *compile* time, not at the point of assignment.

```python
x = 10
def f():
    print(x)   # UnboundLocalError, not 10
    x = 20     # this line, below, made x local everywhere in f
```

The compiler sees `x = 20`, marks `x` as a local variable of `f`, and emits `LOAD_FAST` for the `print(x)` — which finds an unbound slot. This is not a lookup that "falls back" to global; local names are resolved by array slot, not dictionary lookup, which is precisely why Python function locals are fast.

To rebind a name in an outer scope you must declare intent:

```python
counter = 0
def bump():
    global counter      # rebind the module-level name
    counter += 1

def outer():
    n = 0
    def inner():
        nonlocal n      # rebind outer()'s n, NOT global
        n += 1
    inner()
    return n            # 1
```

`global` in application code is nearly always a design smell — it makes functions untestable and non-thread-safe. `nonlocal` is legitimate but rare; if you need much of it, you probably want a class or a `dict`/`list` accumulator (mutation needs no declaration — only *rebinding* does).

```python
def outer():
    seen = []           # no nonlocal needed:
    def inner(x):
        seen.append(x)  # this MUTATES, it does not REBIND
    ...
```

**Comprehensions have their own scope** (Python 3). The loop variable does not leak, and a comprehension at class scope cannot see class-level names beyond the first iterable — a genuine gotcha (§9).

### 2.4 Closures

A **closure** is a function that references a name from an enclosing function's scope. Python implements this with *cells*: the inner function keeps a `__closure__` tuple of cell objects, each holding a reference to a live variable.

```python
def make_multiplier(factor):
    def multiply(x):
        return x * factor      # 'factor' is a free variable
    return multiply

triple = make_multiplier(3)
triple(5)                       # 15

triple.__closure__              # (<cell at 0x...: int object at 0x...>,)
triple.__closure__[0].cell_contents   # 3
triple.__code__.co_freevars     # ('factor',)
```

The enclosing call has returned, yet `factor` is alive — the cell keeps it alive. That is both the power (state without a class) and the hazard (a closure can pin a large object in memory indefinitely).

**Late binding — the trap.** Closures capture *variables*, not *values*:

```python
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]            # [2, 2, 2]  — NOT [0, 1, 2]
```

All three lambdas close over the same `i`, which is `2` by the time they run. Fix by binding at definition time with a default argument (defaults *are* evaluated eagerly):

```python
funcs = [lambda i=i: i for i in range(3)]
[f() for f in funcs]            # [0, 1, 2]

# or, more explicitly:
from functools import partial
funcs = [partial(lambda i: i, i) for i in range(3)]
```

You will hit this for real when building callbacks in a loop — button handlers, retry wrappers, per-partition Kafka consumers. Recognising it on sight is an interview marker.

### 2.5 Decorators

A decorator is a callable that takes a function and returns a replacement.

```python
@log_calls
def work(): ...

# is EXACTLY:
def work(): ...
work = log_calls(work)
```

**The canonical template — memorise this shape:**

```python
import functools

def log_calls(func):
    @functools.wraps(func)                 # copies __name__, __doc__, __wrapped__, __dict__...
    def wrapper(*args, **kwargs):
        print(f"→ {func.__name__}({args}, {kwargs})")
        result = func(*args, **kwargs)     # do NOT forget to return this
        print(f"← {func.__name__} = {result!r}")
        return result
    return wrapper
```

`functools.wraps` is not cosmetic. Without it, `work.__name__` becomes `"wrapper"`, the docstring vanishes, Sphinx documents nothing, pytest reports garbage names, and `inspect.signature` — which **FastAPI and pytest depend on** — returns `(*args, **kwargs)` instead of the real signature. A decorator without `wraps` will silently break FastAPI's dependency injection. `wraps` also sets `__wrapped__`, which is how `inspect.signature` follows the chain back to the original.

**Parameterised decorators** need one more layer — a *decorator factory*:

```python
def retry(times=3, delay=0.5, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last = None
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    last = exc
                    if attempt < times:
                        time.sleep(delay * 2 ** (attempt - 1))   # exponential backoff
            raise last
        return wrapper
    return decorator

@retry(times=5, exceptions=(ConnectionError,))
def fetch(url): ...
```

Read the layers outward: `retry(times=5)` runs first and returns `decorator`; `decorator(fetch)` runs next and returns `wrapper`; the name `fetch` is bound to `wrapper`. Note the trap: `@retry` without parentheses passes the *function* as `times`. Handling both forms requires a sentinel (see Homework Hard).

**Class-based decorators** are natural when the decorator has real state or a public API:

```python
class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)   # the wraps equivalent for classes
        self.func = func
        self.count = 0
    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.func(*args, **kwargs)

@CountCalls
def ping(): ...

ping(); ping()
ping.count          # 2  <- a queryable attribute; hard to do with a plain closure
```

Caveat: this breaks on methods, because the instance is not a descriptor and won't bind `self`. Fix by implementing `__get__` — or just use a closure.

**Stacking** applies bottom-up:

```python
@app.route("/x")     # applied 3rd (sees the authed+timed function)
@requires_auth       # applied 2nd
@timed               # applied 1st (innermost, closest to the real function)
def handler(): ...
# handler = app.route("/x")(requires_auth(timed(handler)))
```

Order matters enormously. `@timed` innermost measures only the handler; outermost it measures auth too. Registration decorators (`@app.route`) go on top so they register the *fully wrapped* callable.

### 2.6 The `functools` trio you must know cold

```python
from functools import wraps, lru_cache, cache, partial, singledispatch

@lru_cache(maxsize=128)          # bounded LRU; @cache is lru_cache(maxsize=None), unbounded
def expensive(n): ...
expensive.cache_info()           # CacheInfo(hits=.., misses=.., maxsize=128, currsize=..)
expensive.cache_clear()

connect_local = partial(connect, "localhost", port=5432)   # freeze arguments
```

`lru_cache` requires **hashable** arguments (no lists/dicts), keys on the *exact* call form (`f(1)` and `f(x=1)` are different keys!), and holds **strong references** to both arguments and results. On a method, `self` becomes part of the key — so every instance ever passed is retained for the process lifetime. That's a textbook production memory leak, and a great interview answer.

---

## 3. Official documentation quotes

> "When one or more top-level parameters have the form *parameter* `=` *expression*, the function is said to have 'default parameter values.' … **Default parameter values are evaluated from left to right when the function definition is executed.** This means that the expression is evaluated once, when the function is defined, and that the same 'pre-computed' value is used for each call."
> — *The Python Language Reference*, [8.7. Function definitions](https://docs.python.org/3/reference/compound_stmts.html#function-definitions)

> "If a name is bound in a block, it is a local variable of that block, unless declared as `nonlocal` or `global`. … The `global` statement has the same scope as a name binding operation in the same block."
> — *The Python Language Reference*, [4.2.2. Resolution of names](https://docs.python.org/3/reference/executionmodel.html#resolution-of-names)

> "The `nonlocal` statement causes the listed identifiers to refer to previously bound variables in the nearest enclosing scope excluding globals."
> — *The Python Language Reference*, [7.13. The nonlocal statement](https://docs.python.org/3/reference/simple_stmts.html#the-nonlocal-statement)

> "`functools.wraps(wrapped)` — This is a convenience function for invoking `update_wrapper()` as a function decorator when defining a wrapper function. … Without the use of this decorator factory, the name of the example function would have been `'wrapper'`, and the docstring of the original `example()` would have been lost."
> — *Python Standard Library*, [`functools`](https://docs.python.org/3/library/functools.html#functools.wraps)

> "Since a dictionary is used to cache results, the positional and keyword arguments to the function must be hashable."
> — *Python Standard Library*, [`functools.lru_cache`](https://docs.python.org/3/library/functools.html#functools.lru_cache)

---

## 4. Real-world examples

### 4.1 A timing decorator that logs properly (not `print`)

```python
import functools, logging, time

log = logging.getLogger(__name__)

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()          # perf_counter, NOT time.time()
        try:
            return func(*args, **kwargs)
        finally:                             # finally => timing survives exceptions
            elapsed = time.perf_counter() - start
            log.info("%s took %.3f ms", func.__qualname__, elapsed * 1000)
    return wrapper

@timed
def slow_query(n):
    time.sleep(0.05)
    return n * 2

logging.basicConfig(level=logging.INFO)
slow_query(21)
```

Expected output:
```
INFO:__main__:slow_query took 50.«…» ms
```

Two senior details: `perf_counter` is a monotonic high-resolution clock (`time.time()` can go *backwards* on NTP adjustment, yielding negative durations in your metrics), and the `finally` ensures failed calls are still measured — which is exactly when you most want the number.

### 4.2 A registry: functions as first-class values

```python
HANDLERS = {}

def handles(event_type):
    def decorator(func):
        HANDLERS[event_type] = func
        return func                    # NOTE: return the original, unwrapped
    return decorator

@handles("user.created")
def on_user_created(payload):
    return f"welcome {payload['name']}"

@handles("user.deleted")
def on_user_deleted(payload):
    return f"goodbye {payload['name']}"

def dispatch(event):
    handler = HANDLERS.get(event["type"])
    if handler is None:
        raise ValueError(f"no handler for {event['type']}")
    return handler(event["payload"])

print(dispatch({"type": "user.created", "payload": {"name": "Ann"}}))
```
```
welcome Ann
```

This is the entire architecture of Flask routing, Celery task registration, and most Kafka consumer frameworks. Note the decorator returns `func` unchanged — it has a **side effect** (registration) rather than wrapping. That is a completely legitimate decorator pattern and a common interview follow-up.

### 4.3 Closure-based rate limiter (state without a class)

```python
import time

def rate_limited(calls_per_second):
    min_interval = 1.0 / calls_per_second
    def decorator(func):
        last_called = 0.0                    # captured by the closure
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            nonlocal last_called             # rebinding => nonlocal required
            wait = min_interval - (time.perf_counter() - last_called)
            if wait > 0:
                time.sleep(wait)
            last_called = time.perf_counter()
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limited(calls_per_second=2)
def call_api(i):
    print(f"{time.strftime('%H:%M:%S')} call {i}")

for i in range(3):
    call_api(i)
```
```
14:22:01 call 0
14:22:01 call 1     <- ~0.5s later
14:22:02 call 2     <- ~0.5s later
```

This is *not* thread-safe — two threads can both read `last_called` before either writes. Fixing it needs a `threading.Lock` in the closure. Spotting that unprompted is a strong signal in an interview.

### 4.4 `lru_cache` on a recursive function

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

fib(100)            # instant — 2^100 calls collapse to 101
fib.cache_info()    # CacheInfo(hits=98, misses=101, maxsize=None, currsize=101)
```

Without the cache, `fib(35)` takes seconds and `fib(100)` never finishes. With it, the recursion is linear. Note `fib(100)` returns a Python `int` of arbitrary precision — no overflow, unlike C or Java.

---

## 5. How This Is Used In Production

- **Startups:** decorators are the fastest way to add cross-cutting behaviour without touching business logic — `@retry` on flaky third-party APIs, `@timed` for a first look at latency, `@lru_cache` on config lookups and reference-data reads. Small teams lean hard on `@app.get(...)`/`@task` style declarative registration because it keeps the codebase legible with few people.
- **Enterprises:** the same mechanism becomes governed infrastructure. A platform team ships an internal library of decorators — `@traced` (OpenTelemetry span), `@authorized(scope="billing:write")`, `@feature_flag("new-pricing")`, `@circuit_breaker`, `@idempotent(key=...)` — and product teams compose them. This is how policy is enforced consistently across hundreds of services without code review catching every case.
- **Common architectures:** the middleware/interceptor stack is decorators at the framework level (ASGI middleware in FastAPI/Starlette is literally a chain of wrapped callables). Airflow's TaskFlow API turns `@task`-decorated functions into DAG nodes and infers dependencies from call sites. pytest's `@pytest.fixture` + name-based injection is a closure registry.
- **Scaling:** `@lru_cache` in front of a hot pure function is often the cheapest order-of-magnitude win available — but on multi-process WSGI/ASGI deployments (Gunicorn with N workers) each process has its **own** cache, so hit rates divide by N and memory multiplies by N. That's the moment you graduate to Redis (see the `fastapi-surwill` track, Lesson 16). Caching decorators also need per-key TTLs and stampede protection at scale; naive `lru_cache` has neither.
- **Monitoring & logging:** decorators are where structured logging and metrics get attached — increment a Prometheus counter, record a histogram of `perf_counter` deltas, open a tracing span, attach `request_id` from a `ContextVar`. Because the decorator sees `func.__qualname__`, you get consistent metric names for free.
- **Security:** authentication/authorization decorators are pervasive (`@login_required` in Django, `Depends(get_current_user)` in FastAPI). The critical rule: the decorator must **fail closed** — any exception in the auth check must deny, never fall through to the wrapped function. A `try/except` that swallows and continues is a real CVE pattern.
- **CI/CD:** decorators are load-bearing for test suites — `@pytest.mark.parametrize`, `@pytest.mark.skipif(sys.platform == "win32")`, `@freeze_time`. `@functools.wraps` matters here specifically: without it, parametrised test IDs and `--collect-only` output become unreadable.
- **Common tools:** `functools` (stdlib), `tenacity` (retry, far better than hand-rolled), `cachetools` (TTL/LFU caches `lru_cache` can't do), `wrapt` (Graham Dumpleton's library — the *correct* way to write decorators that work on methods, classmethods and descriptors), `decorator` (Micheli).
- **Real examples:** **Netflix**'s Python tooling and **Uber**'s data platform wrap task functions to attach lineage and retries. **Google**'s and **Dropbox**'s internal Python style guides both explicitly warn about mutable default arguments and closure late binding, because both have caused production incidents. **Cloudflare** and **Stripe** expose retry/backoff idioms that map directly onto the `@retry` shape above; Stripe's idempotency-key design is the distributed version of `@lru_cache`.

---

## 6. Advanced corner

**Locals are array slots, not dict entries.** CPython compiles function locals to fixed indices and emits `LOAD_FAST`/`STORE_FAST` (an array index) instead of `LOAD_NAME` (a dict lookup). This is why locals are ~2× faster than globals and why the micro-optimisation `_len = len` at the top of a hot loop exists in stdlib code. Verify with `dis`:

```python
import dis
def f():
    x = 1
    return x
dis.dis(f)     # LOAD_CONST / STORE_FAST / LOAD_FAST — no dict lookup anywhere
```

It's also *why* `UnboundLocalError` exists as a distinct error from `NameError`: the slot exists, it's just empty. (In 3.11+ the interpreter is "adaptive" and specialises these opcodes further; in 3.12+ `LOAD_FAST` variants distinguish whether a null check is needed.)

**The cell indirection.** A closed-over variable is stored in a `cell` object, and the enclosing function uses `MAKE_CELL`/`LOAD_DEREF` rather than `LOAD_FAST`. This is why closures are marginally slower than locals, and why two closures created in the same call *share* state — they share the cell. That's how `rate_limited` above works, and it's how you can build a pair of `getter`/`setter` closures over one variable.

**Decorators are a lie about identity.** After `@d`, the name no longer refers to your function. Consequences that bite:
- `isinstance`/`is` checks against the original fail.
- Pickling a decorated function fails unless the wrapper is importable at module level (this breaks `multiprocessing` — a classic "works locally, dies in prod" bug).
- Stack traces gain a frame per decorator; a five-deep decorator stack makes tracebacks genuinely hard to read.
- `inspect.signature` follows `__wrapped__` (set by `wraps`) — but `inspect.getsource` does not, so debuggers may show you the wrapper.

**Decorators on methods are subtly different.** A plain closure decorator works because the wrapper is a function and functions are descriptors — `self` binds normally. A *class-based* decorator is not a descriptor, so `@CountCalls` on a method silently breaks. `wrapt` solves this properly by proxying `__get__`. Know this; it's a great senior answer.

**`lru_cache` failure modes, ranked by how often they cause incidents:**
1. **Unbounded `@cache` on a function with high-cardinality input** — memory grows without limit. Always set `maxsize` unless the key space is provably small.
2. **On a method** — `self` is in the key, instances never die.
3. **Cached mutable return values** — the caller mutates the cached list and poisons every future hit. Return tuples/frozensets, or copy.
4. **No TTL** — cached stale reference data forever. `lru_cache` cannot expire; use `cachetools.TTLCache` or Redis.
5. **Not thread-safe under all conditions** — `lru_cache` *is* thread-safe for cache bookkeeping, but the wrapped function can still be invoked concurrently for the same missing key (no stampede protection).

**What most people get wrong at this level:**
- Believing `global` and `nonlocal` are about *reading* outer variables. They aren't — reading always works. They're purely about **rebinding**.
- Thinking mutable default arguments are a bug in Python. They're a direct consequence of "`def` executes once," which is the same rule that makes decorators and closures work. It's coherent, not accidental.
- Writing decorators without `wraps` and then wondering why FastAPI's DI or pytest's parametrize broke.
- Reaching for a class when a closure is clearer, or a closure when the state deserves a class. Rule of thumb: **one piece of state and one operation → closure; multiple operations or introspectable state → class.**

---

## 7. Best practices

- **Always `@functools.wraps`.** No exceptions. It costs one line and preserves the introspection your frameworks depend on.
- **Never use a mutable default.** Use `None` as the sentinel and build inside: `def f(items=None): items = items or []` — but beware, `or` also replaces an intentionally-passed empty list; prefer `if items is None: items = []`.
- **Make booleans and tuning parameters keyword-only** with a bare `*`. It's the cheapest readability win in the language.
- **Return the result from your wrapper.** Forgetting `return func(...)` turns every decorated function into one that returns `None` — a bug that passes syntax checks and slips through review.
- Use `finally` in decorators that measure or clean up, so exceptions don't skip the bookkeeping.
- Prefer `tenacity` over a hand-rolled `@retry` in production — it has jitter, backoff strategies, and stop conditions you'd otherwise reinvent badly.
- **Bound your caches.** `@lru_cache(maxsize=1024)`, not `@cache`, unless you can prove the key space is small and static.
- Type-annotate decorators with `ParamSpec` so type checkers see through them (§ below) — otherwise mypy erases every decorated signature.
- Keep decorator stacks shallow (2–3). Beyond that, tracebacks and reasoning both degrade.

```python
from typing import ParamSpec, TypeVar, Callable
P = ParamSpec("P"); R = TypeVar("R")

def timed(func: Callable[P, R]) -> Callable[P, R]:     # signature preserved for mypy
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return func(*args, **kwargs)
    return wrapper
```

## 8. Common mistakes & gotchas

**The mutable default argument**
```python
def append_to(item, target=[]):      # BUG: one list, shared by every call, forever
    target.append(item)
    return target

append_to(1)    # [1]
append_to(2)    # [1, 2]   <- surprise
```
`target=[]` was evaluated **once**, at `def` time, and lives in `__defaults__`. The fix:
```python
def append_to(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```
This applies to `{}`, `set()`, `datetime.now()` (frozen at import time!), and any function call in a default.

**Late binding in loops** — covered in §2.4. `[lambda: i for i in range(3)]` gives `[2, 2, 2]`.

**`UnboundLocalError`** — assigning anywhere makes the name local everywhere in that function.

**Comprehension scope at class level**
```python
class C:
    names = ["a", "b"]
    upper = [n.upper() for n in names]          # OK — first iterable evaluated in class scope
    bad   = [n + sep for n in names]            # NameError if sep is a class attribute
    sep = "-"
```
Class bodies are not enclosing scopes for comprehensions. Move it to `__init__` or use a module-level constant.

**Forgetting to return from the wrapper** — silently returns `None`.

**`@retry` vs `@retry()`** — without parentheses, the decorated function is bound to `times`.

**Decorating a method with a class-based decorator** — `self` won't bind.

**Mutating a cached return value** — `lru_cache` hands out the same object every hit.

## 9. Where AI helps (and where it hurts)

- **Helps:** generating the boilerplate for a parameterised decorator (the three-layer nesting is fiddly and easy to typo); explaining an unfamiliar decorator stack in an existing codebase; writing the `ParamSpec` annotations correctly; converting a hand-rolled retry loop to `tenacity`; producing `dis` walkthroughs for study.
- **Hurts:** AI-generated decorators very frequently **omit `functools.wraps`** and sometimes **forget the `return`**, both of which produce code that runs fine in a smoke test and fails weirdly in a framework. It will also cheerfully suggest `@lru_cache` on methods and on functions taking unhashable args, and it rarely mentions the memory/TTL implications. It's confidently wrong about *when* defaults are evaluated in explanations roughly as often as it's right. Treat every generated decorator as needing a `wraps` audit and a "does it return?" audit before you commit.

## 10. Learn independently

**Books (specific chapters):**
- *Fluent Python*, 2nd ed. (Ramalho) — **Ch. 7 "Functions as First-Class Objects"**, **Ch. 8 "Type Hints in Functions"** (`ParamSpec`, callable types), and **Ch. 9 "Decorators and Closures"** — the definitive treatment, including the `registration` pattern and late binding. Read Ch. 9 twice.
- *Effective Python*, 2nd ed. (Slatkin) — **Item 20** (raise exceptions rather than return `None`), **Item 21** ("Know How Closures Interact with Variable Scope" — the canonical `nonlocal` discussion), **Item 22–25** (`*args`, keyword-only, `None`-as-default), **Item 26** ("Define Function Decorators with `functools.wraps`").
- *Python Cookbook*, 3rd ed. (Beazley & Jones) — **Ch. 7 "Functions"** and **Ch. 9 "Metaprogramming"** — recipes 9.1–9.12 cover decorators with arguments, class decorators, decorators on methods, and signature preservation with `inspect.Signature`. The most practical decorator reference in print.
- *CPython Internals* (Shaw) — **Ch. "The Evaluation Loop"** and the frame/code-object chapters — explains `LOAD_FAST` vs `LOAD_DEREF` and what a cell actually is.
- *Robust Python* (Viafore) — **Ch. 3–4** on annotating callables and why erased signatures hurt.

**Primary docs:**
- [Execution model — resolution of names](https://docs.python.org/3/reference/executionmodel.html#resolution-of-names) — the normative LEGB text.
- [Function definitions](https://docs.python.org/3/reference/compound_stmts.html#function-definitions) — parameter grammar, `/` and `*`.
- [`functools`](https://docs.python.org/3/library/functools.html) — read `wraps`, `lru_cache`, `partial`, `singledispatch`, `cached_property` in full.
- [`inspect`](https://docs.python.org/3/library/inspect.html#inspect.signature) — `signature`, `__wrapped__` unwrapping.
- [PEP 318](https://peps.python.org/pep-0318/) (decorators), [PEP 3102](https://peps.python.org/pep-3102/) (keyword-only), [PEP 570](https://peps.python.org/pep-0570/) (positional-only), [PEP 612](https://peps.python.org/pep-0612/) (`ParamSpec`).

**High-quality links:**
- [Primer on Python Decorators](https://realpython.com/primer-on-python-decorators/) — the best free long-form walkthrough.
- [`wrapt` documentation](https://wrapt.readthedocs.io/) — Graham Dumpleton's series "How you implemented your Python decorator is wrong" is required reading if you ship a decorator library.
- [`tenacity`](https://tenacity.readthedocs.io/) — study its API as a design exemplar for parameterised decorators.

## 11. Interview preparation

**Q1. What does the `@` syntax actually do?**
> It's syntactic sugar for reassignment: `@d` above `def f` compiles to `f = d(f)` executed immediately after the `def`. The decorator is any callable taking the function and returning a replacement — usually a wrapper closure, but it can return the original unchanged (registration decorators) or something else entirely (`property` returns a descriptor object). There's no special runtime machinery.

**Q2. Explain the mutable default argument problem, and why Python is designed this way.**
> Default values are evaluated once, when the `def` statement executes, and stored in `func.__defaults__`. A mutable default is therefore a single object shared across all calls, so mutations persist between them. Use `None` as a sentinel and construct inside the body. It isn't a wart so much as a consequence of "`def` is an executable statement that builds a function object" — the same rule that makes decorators, closures, and function factories possible. The alternative (evaluating defaults per call) would need to re-run arbitrary expressions on every call, which is both slower and, for expressions with side effects, more surprising.

**Q3. What's the difference between `global` and `nonlocal`?**
> Both are about **rebinding**, not reading — reading an outer name always works via LEGB. `global x` binds `x` in the module namespace; `nonlocal x` binds `x` in the nearest enclosing *function* scope, and errors at compile time if no such binding exists. Neither is needed to mutate a mutable object — `list.append` on an outer list requires no declaration, because it doesn't rebind the name.

**Q4. Why does `[lambda: i for i in range(3)]` return `[2, 2, 2]`?**
> Closures capture the variable (a cell), not the value. All three lambdas share the same `i` cell; by the time any of them is called, the comprehension has finished and `i` is `2`. Bind eagerly with a default argument (`lambda i=i: i`) or `functools.partial`, since defaults *are* evaluated at definition time. This bites in real code whenever you build callbacks or handlers in a loop.

**Q5. Why is `functools.wraps` necessary?**
> Without it, the decorated name refers to `wrapper`, so `__name__`, `__doc__`, `__module__`, `__qualname__`, `__dict__`, and `__annotations__` all describe the wrapper rather than the real function. That breaks documentation tooling, makes logs and tracebacks say "wrapper," and — critically — makes `inspect.signature` report `(*args, **kwargs)`. Frameworks that introspect signatures (FastAPI's dependency injection, pytest's fixture matching, Click, Typer) will misbehave or fail. `wraps` also sets `__wrapped__`, letting `inspect.signature` unwrap to the original.

**Q6. Write a decorator that retries with exponential backoff. What would you actually use in production?**
> [Give the `@retry` implementation from §2.5.] In production I'd use **`tenacity`** rather than hand-rolling it, because it provides jitter (essential — synchronised retries from many clients cause a thundering herd that turns a blip into an outage), pluggable stop/wait strategies, retry-on-result predicates, and async support. Hand-rolled retries also tend to miss two things: only retrying *idempotent* operations, and respecting a total deadline rather than just a retry count.

**Q7. (System design) You have `@lru_cache` on a hot function in a FastAPI service running under Gunicorn with 8 workers, on 4 pods. What's wrong?**
> The cache is per-process, so there are 32 independent caches. Three problems follow. **Correctness:** the caches can disagree — a user's request can hit a stale entry on one pod and fresh data on another, producing non-deterministic responses. **Memory:** cache size multiplies by 32; a "500 MB cache" is now 16 GB of RSS and your pods OOM. **Effectiveness:** hit rate collapses because each cache sees 1/32 of the traffic, and cold starts/rollouts wipe everything. The fix is a shared cache — Redis — with explicit TTLs and invalidation, accepting the network hop in exchange for coherence and bounded memory. Keep `lru_cache` only for genuinely immutable, small, process-local data (parsed config, compiled regexes, reference tables). If you need both, do a two-tier cache: tiny in-process LRU with a short TTL in front of Redis, and accept the bounded staleness explicitly.

**Q8. (Trade-off) Closure vs class for stateful behaviour — how do you choose?**
> Closures are lighter and keep state genuinely private (reachable only via `__closure__` gymnastics), which suits a single piece of state and a single operation — a counter, a rate limiter, a memoiser. Classes win when there are multiple operations over the same state, when the state should be introspectable or resettable (`ping.count`, `cache_clear()`), when you want to subclass or inject the behaviour, or when pickling matters (closures don't pickle; `multiprocessing` will fail). Classes are also easier to test in isolation. My default: closure until I need a second method or want to see the state from outside, then class.

**Q9. What is `functools.partial` and how does it differ from a lambda?**
> `partial(f, a, b=1)` returns a callable with those arguments frozen. Versus `lambda *a, **k: f(x, *a, **k)`: `partial` binds eagerly (no late-binding surprise), is picklable (works with `multiprocessing`), is introspectable (`.func`, `.args`, `.keywords`), and is implemented in C so it's faster. A lambda is more flexible when you need to *transform* arguments rather than just freeze them. In a loop, prefer `partial` — it sidesteps the late-binding bug entirely.

**Q10. How do you preserve type information through a decorator?**
> Use `ParamSpec` (PEP 612, 3.10+): declare `P = ParamSpec("P")`, `R = TypeVar("R")` and type the decorator `Callable[P, R] -> Callable[P, R]`, with the wrapper taking `*args: P.args, **kwargs: P.kwargs`. Without this, mypy sees `(*args: Any, **kwargs: Any) -> Any` and every decorated call site loses type checking silently — you keep the annotations at runtime but lose all static safety, which is the worst of both worlds because it *looks* typed.

**Q11. What does the decorator order `@a @b def f` mean, and when does order matter?**
> `f = a(b(f))` — bottom-up application, so `b` is innermost/closest to `f`, and at call time `a`'s wrapper runs first. Order matters whenever the decorators are not commutative: a `@timed` inside `@retry` measures each attempt, outside it measures total elapsed including sleeps. `@cache` outside `@authorize` would serve cached data to unauthorized users — a security bug. Registration decorators (`@app.route`) must be outermost so the fully-wrapped callable is what gets registered.

**Q12. (Senior) How would you write a decorator that works correctly on functions, methods, classmethods, and staticmethods?**
> A plain closure decorator already works on instance methods because functions are descriptors — but it breaks when applied *above* `@classmethod`/`@staticmethod` (those objects aren't callable in older versions and don't forward correctly), and class-based decorators break on methods because instances aren't descriptors. Doing it properly means implementing `__get__` on your wrapper to bind correctly, or using **`wrapt`**, whose `@wrapt.decorator` provides a transparent object proxy that handles all four cases plus `isinstance` checks and attribute forwarding. The pragmatic answer: apply decorators *below* `@classmethod`/`@staticmethod`, use closures not classes, and reach for `wrapt` if you're shipping a library.

## 12. Homework

> Work in `python/solutions/lesson-04/`. Every task must run on Python 3.12+. Add a short `README.md` per task explaining your design choice.

**Easy**

1. **Binding grammar.** Write `def report(name, /, level="INFO", *tags, verbose=False, **extra)`. Then write five call sites: one valid using every section; one that raises `TypeError` for passing `name` by keyword; one that raises for passing `verbose` positionally; one using `*`/`**` unpacking from a tuple and dict; one that puts a key called `name` into `**extra` (and explain in a comment why this works only because of `/`).
   *Acceptance:* a script printing the bound values for the valid calls and catching + printing each `TypeError` message.

2. **The default-argument trap.** Write `buggy_append(item, target=[])` and demonstrate the shared-state bug across three calls. Then write `fixed_append` with the `None` sentinel. Also demonstrate the *time* variant: `def stamp(t=datetime.now())` — show that `t` is frozen at import, not per call.
   *Acceptance:* output shows the growing list, the correct isolated lists, and two identical timestamps from calls seconds apart.

3. **Scope drill.** Without running it, predict the output of five snippets covering: `UnboundLocalError`, `global`, `nonlocal`, mutation-without-declaration, and a comprehension not leaking its loop variable. Write your prediction as a comment, *then* run and reconcile any misses in a `NOTES.md`.

**Medium**

4. **`@timed` with percentiles.** Write a decorator that records every call's duration in a closure-held list and exposes `wrapper.stats()` returning `{"count", "mean_ms", "p50_ms", "p95_ms", "max_ms"}`. Use `time.perf_counter` and a `finally` so failed calls are still timed. Add `wrapper.reset()`.
   *Acceptance:* a demo function with randomised `sleep`; `stats()` returns plausible percentiles; a call that raises still increments `count`.

5. **`@retry` done properly.** Implement `retry(times=3, base_delay=0.1, max_delay=5.0, exceptions=(Exception,), jitter=True)` with exponential backoff and full jitter (`sleep(random.uniform(0, min(max_delay, base_delay * 2**attempt)))`). Re-raise the last exception if all attempts fail. Log each attempt via `logging` at WARNING.
   *Acceptance:* a flaky function failing twice then succeeding returns correctly with 3 attempts logged; a permanently-failing function raises the *original* exception type after exactly `times` attempts; sleeps are bounded by `max_delay`.

6. **Registry + dispatch.** Build an event system: `@handles("order.placed")` registers handlers into a module-level dict, allowing **multiple** handlers per event (a list). Add `dispatch(event)` that calls all handlers, collects results, and does not let one failing handler prevent the others (log and continue). Add `handlers_for(event_type)` and a `clear_registry()` for tests.
   *Acceptance:* pytest tests proving multi-handler fan-out, isolation on failure, and that an unknown event type raises a clear `ValueError`.

7. **Late binding, three fixes.** Build a list of 5 callbacks that should print `0..4`. Show the broken version, then fix it three ways: default-argument binding, `functools.partial`, and a factory function returning a closure. Write a paragraph in `NOTES.md` on which you'd use in review and why.

**Hard**

8. **A decorator that works with and without arguments.** Implement `@trace` such that all three forms work identically:
   ```python
   @trace
   def a(): ...
   @trace()
   def b(): ...
   @trace(level="DEBUG")
   def c(): ...
   ```
   Use a sentinel and `functools.partial`. Must preserve `__name__`, `__doc__`, and `inspect.signature`.
   *Acceptance:* pytest asserting `inspect.signature(c) == inspect.signature(<original c>)`, `c.__name__ == "c"`, `c.__wrapped__ is <original>`, and correct behaviour for all three forms.

9. **A caching decorator `lru_cache` can't be.** Implement `@memoize(maxsize=128, ttl=60, typed=False, key=None)` with: LRU eviction (use `collections.OrderedDict` or a dict + `move_to_end`), per-entry TTL expiry, an optional custom `key` callable so **unhashable** arguments (dicts, lists) can be cached by a derived key, thread safety via `threading.Lock`, and `cache_info()` / `cache_clear()` matching the `lru_cache` API.
   *Acceptance:* pytest covering — eviction order under `maxsize=2`; expiry using a monkeypatched clock (do **not** `sleep`); a `dict` argument cached via `key=lambda d: tuple(sorted(d.items()))`; `cache_info()` hit/miss counts; and a concurrency test with 10 threads hammering one key that never corrupts the counters.

10. **Type-safe decorator stack.** Take tasks 4, 5 and 8, annotate all three with `ParamSpec`/`TypeVar`, and prove with `mypy --strict` that a decorated `def add(a: int, b: int) -> int` still rejects `add("x", 1)` at type-check time. Include the failing mypy output in `NOTES.md`.
    *Acceptance:* `mypy --strict` passes on the module; a deliberate bad call site in a `# type: ignore`-free file produces the expected error.

**Stretch (senior)**

11. **Break, then fix, a method decorator.** Write a class-based `@CountCalls` and apply it to (a) a plain function, (b) an instance method, (c) a `@classmethod`, (d) a `@staticmethod`. Document precisely which break and *why* (descriptor protocol). Then fix it by implementing `__get__` on the decorator class. Finally, reimplement the same thing with `wrapt.decorator` in ~5 lines and compare.
    *Acceptance:* a test matrix showing all four cases passing for both the `__get__` version and the `wrapt` version, plus a written explanation of the descriptor binding.

12. **Prove the `lru_cache` leak.** Build a class holding a 10 MB `bytes` payload, put `@lru_cache` on one of its methods, create 100 instances, drop all references, force `gc.collect()`, and measure RSS (`tracemalloc` or `psutil`). Show the memory is retained. Then fix it three ways — `cached_property`, a per-instance cache in `__init__`, and `weakref` — and measure each.
    *Acceptance:* a table of measured RSS before/after for all four variants, and a recommendation with justification.

## 13. Key takeaways

- `def` **executes**; it builds a function object with a code object, `__defaults__` (evaluated **once**), `__closure__`, and `__globals__`. Nearly every gotcha in this lesson follows from that one sentence.
- Argument binding has five sections (`pos, /, standard, *args, kw_only, **kwargs`). Use `*` to force booleans and knobs to be keyword-only — the cheapest API win available.
- LEGB resolves names; **assignment anywhere makes a name local everywhere** in that function. `global`/`nonlocal` are about *rebinding*, never about reading.
- Closures capture **variables (cells), not values** — hence late binding. Bind eagerly with a default argument or `partial`.
- `@d` is exactly `f = d(f)`. Parameterised decorators are three layers deep; stacking applies bottom-up and order is semantically significant.
- **Always `functools.wraps`** — without it you break `inspect.signature`, and with it you break FastAPI, pytest, Click, and your own tracebacks.
- `lru_cache` is a scalpel: bound it, keep it off methods, never cache mutable returns, and know that it doesn't survive contact with multi-process deployments.
