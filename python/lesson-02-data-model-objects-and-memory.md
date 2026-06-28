# Python Lesson 02 — Data Model, Objects & Memory

| | |
|---|---|
| **Track** | Python |
| **Lesson** | 02 of 15 |
| **Topic** | How Python represents data: names vs objects, mutability, identity, and references |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lesson 01 (you can run Python in a venv) |

### Learning objectives
1. Explain the difference between a **name (variable)** and an **object**.
2. Predict whether two names share the same object (`is` vs `==`).
3. Reason confidently about **mutable vs immutable** types and the bugs mutability causes.
4. Understand reference semantics in function arguments ("is Python pass-by-value or by-reference?").
5. Have a working mental model of CPython memory: refcounts, garbage collection, interning.

---

## 1. Why this matters
The single most common class of subtle Python bugs — aliasing, the mutable-default-argument trap, "why did my list change?" — all come from misunderstanding *names vs objects*. This is also the #1 conceptual interview topic for Python. Get the model right once and a whole category of bugs vanishes for the rest of your career.

---

## 2. Theory

### 2.1 Everything is an object; variables are names
In Python, **a variable is a name bound to an object**, not a box holding a value. Assignment binds a name to an object; it never copies the object.

```python
a = [1, 2, 3]
b = a          # b is another NAME for the SAME list object
b.append(4)
print(a)       # [1, 2, 3, 4]  ← a and b alias one object
```

Every object has three things:
- an **identity** (its address in memory — see it with `id(obj)`), never changes during its life;
- a **type** (`type(obj)`), never changes;
- a **value** (may change if the object is mutable).

### 2.2 `is` vs `==`
- `==` asks "do these objects have **equal value**?" (calls `__eq__`).
- `is` asks "are these the **same object**?" (compares identity / `id()`).

```python
x = [1, 2]
y = [1, 2]
x == y      # True  (equal values)
x is y      # False (two different list objects)
```
Rule: use `==` for value comparison; reserve `is` for singletons, especially `is None`.

### 2.3 Mutable vs immutable
- **Immutable** (cannot change after creation): `int`, `float`, `bool`, `str`, `tuple`, `frozenset`, `bytes`. "Modifying" them creates a *new* object.
- **Mutable** (can change in place): `list`, `dict`, `set`, `bytearray`, and most custom class instances.

```python
s = "hello"
print(id(s))
s += " world"      # creates a NEW string; s now names it
print(id(s))       # different id

lst = [1, 2]
print(id(lst))
lst += [3]         # mutates IN PLACE
print(id(lst))     # SAME id
```

This is why `+=` behaves differently for strings (rebind) vs lists (in-place extend).

### 2.4 The classic trap: mutable default arguments
Default argument values are evaluated **once**, when the function is defined — not on each call.

```python
def bad(item, bucket=[]):       # BUG: one shared list for all calls
    bucket.append(item)
    return bucket

bad(1)   # [1]
bad(2)   # [1, 2]  ← surprise! same list

def good(item, bucket=None):    # correct idiom
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

### 2.5 Pass-by-value or pass-by-reference?
Neither label fits cleanly. Python passes **references by value**: the function receives a copy of the *reference* (the name binding), both pointing at the same object.
- If you **mutate** the object, the caller sees it (it's the same object).
- If you **rebind** the parameter to a new object, the caller does *not* see it.

```python
def mutate(d):  d["k"] = 1      # caller sees this
def rebind(d):  d = {"new": 1}  # caller does NOT see this

m = {}
mutate(m); print(m)   # {'k': 1}
rebind(m); print(m)   # {'k': 1}  (unchanged by rebind)
```

### 2.6 Copying: assignment vs shallow vs deep
```python
import copy
original = [[1, 2], [3, 4]]
alias      = original                 # same object
shallow    = original.copy()          # new outer list, SAME inner lists
deep       = copy.deepcopy(original)  # fully independent

original[0].append(99)
print(shallow)  # [[1, 2, 99], [3, 4]] ← inner list was shared
print(deep)     # [[1, 2], [3, 4]]      ← fully isolated
```

### 2.7 CPython memory model (mental model, not implementation detail to memorize)
- CPython uses **reference counting**: each object tracks how many names/containers reference it. When the count hits zero, memory is freed immediately.
- A **cyclic garbage collector** (`gc` module) handles reference cycles (A references B, B references A) that refcounting alone can't free.
- **Interning/caching:** small integers (−5 to 256) and some short strings are cached and reused, so `a is b` may surprisingly be `True` for them. *Never rely on this* — it's an optimization, not a guarantee.

```python
a = 256; b = 256
print(a is b)   # True  (cached)
a = 257; b = 257
print(a is b)   # often False — implementation detail, do not depend on it
```

---

## 3. Official documentation quotes

> "Objects are Python's abstraction for data. All data in a Python program is represented by objects or by relations between objects."
> — *Python Language Reference*, [3. Data model](https://docs.python.org/3/reference/datamodel.html)

> "Every object has an identity, a type and a value. An object's identity never changes once it has been created; you may think of it as the object's address in memory. The `is` operator compares the identity of two objects; the `id()` function returns an integer representing its identity."
> — *Python Language Reference*, [3. Data model](https://docs.python.org/3/reference/datamodel.html)

> "Default parameter values are evaluated from left to right when the function definition is executed. This means that the expression is evaluated once, when the function is defined, and that the same 'pre-computed' value is used for each call."
> — *Python tutorial*, [Default Argument Values](https://docs.python.org/3/tutorial/controlflow.html#default-argument-values)

---

## 4. Real-world examples

### 4.1 Aliasing bug in real code
```python
# A config "template" accidentally shared across users
DEFAULT_PERMS = ["read"]

def make_user(name, perms=DEFAULT_PERMS):  # aliasing the module-level list
    return {"name": name, "perms": perms}

admin = make_user("admin")
admin["perms"].append("write")     # mutates the SHARED list
guest = make_user("guest")
print(guest["perms"])              # ['read', 'write'] ← leaked! security bug
```
Fix: `perms=None` then copy, or `perms or ["read"]`.

### 4.2 Cheap equality vs identity check
```python
def first_or_default(value, default):
    return value if value is not None else default   # 'is not None' is correct
# Using '== None' works but is non-idiomatic and breaks for objects with custom __eq__.
```

---

## 6. How This Is Used In Production
- **Startups:** the mutable-default and aliasing traps cause real incidents — e.g., a shared default `list`/`dict` leaking state between requests in a web handler. Code review and linters (Ruff's `B006`, pylint `W0102`) catch these automatically; enabling them is standard.
- **Enterprises:** large codebases lean on **immutability for safety** — passing `tuple`s and `frozenset`s, `@dataclass(frozen=True)` value objects, and defensive copies at API boundaries — because shared mutable state across threads/services is a top source of heisenbugs.
- **Common architectures:** functional-style data pipelines (each stage returns new data instead of mutating) and event-sourcing both lean on immutable data so the same input always yields the same output — crucial for reproducibility in data engineering.
- **Scaling/concurrency:** immutable objects are inherently thread-safe (no one can change them), which matters once you add threads/async (Lesson 13). Shared mutable state is what you must lock.
- **Monitoring/Logging:** memory leaks (refs that never drop to zero — caches, global lists that only grow) are diagnosed with `tracemalloc`, `objgraph`, and heap snapshots. Knowing refcount/GC behavior is how you reason about them.
- **Real examples:** **Instagram** has publicly discussed Python GC tuning at scale (even *disabling* the cyclic GC for certain processes to improve memory copy-on-write behavior in their forked-server model) — a direct, production-grade application of understanding refcounting vs cyclic GC. **Dropbox**'s `mypy` and type adoption is partly about catching these data-model mistakes before runtime.

---

## 7. Best practices
- Use `is`/`is not` only for `None` and other singletons; use `==` for values.
- Never use a mutable object as a default argument — use `None` sentinel.
- Make value objects immutable when you can (`tuple`, `frozenset`, `frozen=True` dataclasses).
- Copy at boundaries: if you store a caller's list/dict, store a copy unless shared mutation is intended.
- Don't depend on int/str interning for correctness.

## 8. Common mistakes & gotchas
- `b = a` does **not** copy a list/dict — it aliases. Use `a.copy()` or `copy.deepcopy`.
- Shallow copy still shares *nested* mutable objects.
- `==` can be expensive or surprising for custom classes; `is` is O(1) identity.
- Mutating a list while iterating over it skips elements/raises — iterate over a copy or build a new list.
- Assuming function arguments are deep-copied — they are not.

## 9. Where AI helps (and where it hurts)
- **Helps:** explaining *why* a value changed unexpectedly (paste the snippet), suggesting the `None`-sentinel fix, generating `__eq__`/`__hash__` correctly, recommending `copy.deepcopy` vs `copy.copy`.
- **Hurts:** AI sometimes "explains" interning as if it were guaranteed (`a is b` for ints) — treat that as an implementation detail, never logic. It can also miss aliasing across modules unless you show it the whole flow.

## 10. Learn independently
- [Python Data Model reference](https://docs.python.org/3/reference/datamodel.html) (dense but foundational).
- *Fluent Python* (Ramalho), ch. 6 "Object References, Mutability, and Recycling."
- Ned Batchelder's talk **"Facts and Myths about Python names and values"** — the canonical explainer.
- [`copy` module docs](https://docs.python.org/3/library/copy.html).

## 11. Interview preparation

**Q1. Is Python pass-by-value or pass-by-reference?**
> Neither exactly — it's "pass by object reference" / "call by sharing." The function gets a copy of the reference, so it points at the same object. Mutating the object is visible to the caller; rebinding the parameter is not.

**Q2. Difference between `is` and `==`?**
> `==` compares value via `__eq__`; `is` compares identity (same object, same `id()`). Use `is` for `None`.

**Q3. Why does this print `[1, 2]`?**
> ```python
> def f(x=[]): x.append(1); return x
> f(); print(f())   # [1, 1]
> ```
> Because the default `[]` is created once at definition time and reused across calls, accumulating state. Fix with a `None` default and create the list inside.

**Q4. Mutable vs immutable — give examples and one consequence.**
> Immutable: `int, str, tuple, frozenset, bytes`. Mutable: `list, dict, set, bytearray`. Consequence: only immutable (hashable) objects can be dict keys / set members; and immutables are safely shareable across threads.

**Q5. How does CPython manage memory?**
> Primarily reference counting — objects are freed when their refcount hits zero — plus a cyclic garbage collector (`gc`) to reclaim reference cycles. Small ints and some strings are interned/cached.

**Q6. What's the difference between shallow and deep copy?**
> Shallow copy duplicates the outer container but shares the nested objects; deep copy recursively duplicates everything so the result is fully independent.

**Q7. Why can a `list` not be a dict key but a `tuple` can?**
> Dict keys must be hashable, and hashability requires immutability (a stable hash). Lists are mutable, so they're unhashable; tuples of hashable items are hashable.

## 12. Homework
> Work in `python/solutions/lesson-02/`.

**Easy**
1. Write `aliasing.py` demonstrating that `b = a` aliases a list (mutating through `b` changes `a`), then fix it so `b` is an independent copy. Print `id()` of both to prove your point.

**Medium**
2. Write `default_trap.py` with a buggy function using a mutable default that leaks state across calls, show the bug with assertions, then implement the corrected `None`-sentinel version and prove (with assertions) it no longer leaks.

**Hard**
3. Write `copy_lab.py` that builds a nested structure (a list of dicts of lists), then creates an alias, a shallow copy, and a deep copy. Mutate a *nested* element of the original and use `assert` statements to demonstrate exactly which of the three change and which don't. Add a short comment explaining each result.

**Stretch:** Use `sys.getrefcount(obj)` and the `gc` module to observe a reference cycle being collected. Create two objects that reference each other, delete the names, force `gc.collect()`, and report what you observe.

## 13. Key takeaways
- Variables are **names bound to objects**; assignment never copies.
- `==` = equal value; `is` = same object (use for `None`).
- Mutable vs immutable drives aliasing bugs, hashability, and thread safety.
- Python is "call by sharing": mutation is visible, rebinding is not.
- Mutable default arguments are a trap — use a `None` sentinel.
- CPython = reference counting + cyclic GC; interning is an optimization, never logic.
