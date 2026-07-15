# Python Lesson 05 — Built-in Data Structures & `collections`

| | |
|---|---|
| **Track** | Python |
| **Lesson** | 05 of 15 |
| **Topic** | `list`, `tuple`, `dict`, `set`, their internals and complexity, and the `collections` module |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–04 (names vs objects; hashability from Lesson 02) |

### Learning objectives
1. Choose the right container by reasoning about **complexity**, not habit — and defend the choice in an interview.
2. Explain how CPython's `list` (dynamic array with over-allocation) and `dict`/`set` (open-addressed hash tables, compact since 3.6) are actually implemented.
3. Explain hashing, the hash/eq contract, and why mutable objects must not be dict keys.
4. Use `defaultdict`, `Counter`, `deque`, `namedtuple`, `OrderedDict`, and `ChainMap` where they genuinely beat the builtins.
5. Reason about memory: `sys.getsizeof`, over-allocation, `__slots__`, and when a list of dicts is the wrong shape entirely.
6. Know when to leave the builtins behind for `heapq`, `bisect`, `array`, or NumPy.

---

## 1. Why this matters

Data structure choice is where Python code silently becomes O(n²). Nobody writes a nested loop on purpose; they write `if item in my_list` inside a `for` and ship it, and six months later a report that took 200 ms takes 40 minutes because the list grew from 100 to 100,000 rows. That is the single most common real performance bug in Python codebases, and it is a one-character fix (`list` → `set`).

For your target roles this is foundational:
- **Data engineering:** every transform is dict/set/list manipulation before it's a DataFrame. Joins are hash-based — `dict`-based lookups *are* hash joins. Deduplication is `set`. Aggregation is `Counter` / `defaultdict(list)`. Understanding the memory profile of a list of a million dicts is the difference between a job that fits in RAM and one that doesn't.
- **Backend:** request routing, caching layers, permission sets, batching queues (`deque`) — all containers.
- **Interviews:** "what's the complexity of `x in list` vs `x in set`?" is a table-stakes screening question. "Why is `dict` ordered now?" and "how does open addressing handle collisions?" separate mid from senior.

---

## 2. Theory

### 2.1 The four builtins at a glance

| | `list` | `tuple` | `dict` | `set` |
|---|---|---|---|---|
| Mutable | yes | **no** | yes | yes |
| Ordered | yes (insertion) | yes | yes (insertion, 3.7+) | **no** |
| Duplicates | yes | yes | keys: no | no |
| Hashable | no | **yes** (if contents are) | no | no (`frozenset` is) |
| Index/lookup | `O(1)` by index | `O(1)` by index | `O(1)` by key | `O(1)` membership |
| Membership `in` | **`O(n)`** | **`O(n)`** | `O(1)` | `O(1)` |
| Append | `O(1)` amortised | — | — | `O(1)` |
| Insert/delete at front | **`O(n)`** | — | — | — |
| Underlying structure | dynamic array | fixed array | hash table | hash table |

The two rows that decide most designs: **membership** and **insert at front**. If you're doing either at scale on a `list`, you've chosen wrong.

### 2.2 `list` internals — a dynamic array of pointers

A CPython `list` is a contiguous array of `PyObject*` pointers plus a length and an allocated capacity. It is **not** a linked list, and it does **not** store objects inline — it stores pointers, which is why a list can hold mixed types and why `sys.getsizeof(list)` doesn't count the elements.

**Over-allocation:** when the array is full, CPython allocates a new one roughly 1.125× larger (the growth pattern is `0, 4, 8, 16, 25, 35, 46, 58, 72, 88…`) and memcpy's the pointers over. Because growth is geometric, `append` is **O(1) amortised** — n appends cost O(n) total, even though individual appends occasionally cost O(n).

```python
import sys
lst = []
prev = sys.getsizeof(lst)
for i in range(20):
    lst.append(i)
    size = sys.getsizeof(lst)
    if size != prev:
        print(f"len={len(lst):2d} capacity grew, size={size} bytes")
        prev = size
```
```
len= 1 capacity grew, size=88 bytes
len= 5 capacity grew, size=120 bytes
len= 9 capacity grew, size=184 bytes
len=17 capacity grew, size=256 bytes
```
(Exact numbers vary by build/version; the *pattern* — jumps, not per-append growth — is the point.)

**Consequences:**
- `list.insert(0, x)` and `list.pop(0)` are **O(n)** — every element shifts. Use `collections.deque` for a queue.
- `del lst[i]` is O(n) for the same reason.
- Lists never shrink their allocation on `pop` until well below capacity — a list that peaked at 10M items holds ~80 MB of pointers until you rebind it.
- `lst.copy()` / `lst[:]` is a **shallow** copy — new array, same pointers.

### 2.3 `dict` internals — compact, open-addressed, insertion-ordered

Since 3.6 (implementation) / 3.7 (guaranteed by the language spec), `dict` is **insertion-ordered**. This wasn't a feature added for ordering — it fell out of a memory optimisation. The layout is:

```
indices:  [-1, 1, -1, 0, -1, -1, 2, -1]        # sparse array of ints into entries
entries:  [ (hash, key, value),                # dense array, in INSERTION order
            (hash, key, value),
            (hash, key, value) ]
```

The sparse `indices` array is small (`int8` while the dict is small, widening as needed); the dense `entries` array holds the actual data in insertion order. Before 3.6, the entries array *was* the sparse one, so ~2/3 of it was empty pointers. The compact layout cut dict memory by 20–25% **and** made iteration order = insertion order as a free side effect.

**Lookup**: `hash(key)` → mask to an index → probe `indices` → follow to `entries` → compare hashes, then `==`. Collisions are resolved by **open addressing** (probe another slot in the same table) rather than chaining (a linked list per bucket). CPython's probe sequence uses the *upper* bits of the hash as it perturbs, which spreads clusters better than linear probing.

**Load factor**: the table resizes (roughly ×3 for small, ×2 for large) when it's 2/3 full, because open addressing degrades badly as it fills. This means a dict's memory jumps in steps, and a resize rehashes everything — O(n), amortised away over the inserts.

**Complexity**: O(1) *average*, **O(n) worst case** if every key collides. In practice `hash()` for strings and ints is good, and string hashing is randomised per-process (PEP 456, SipHash) specifically to prevent adversaries from crafting colliding keys — a real DoS vector on web servers pre-3.3.

### 2.4 Hashing and the hash/eq contract

An object is **hashable** if it has a `__hash__` that doesn't change over its lifetime and an `__eq__` consistent with it. The contract:

> **If `a == b` then `hash(a) == hash(b)`.** (The converse need not hold — equal hashes with unequal objects is just a collision.)

```python
hash("abc")       # varies per process (PYTHONHASHSEED randomisation)
hash(42)          # 42 — small ints hash to themselves
hash(42.0)        # 42 — because 42 == 42.0 MUST imply equal hashes
hash((1, 2))      # fine — tuple of hashables
hash([1, 2])      # TypeError: unhashable type: 'list'
```

Why mutable objects are unhashable: if you could use a list as a key, mutating it would change its hash, and it would be in the wrong bucket — permanently lost inside its own dict. Python forbids the whole class of bug by setting `__hash__ = None` on mutable builtins.

The trap that follows from `42 == 42.0`:
```python
d = {}
d[1] = "int"
d[1.0] = "float"
d[True] = "bool"
d                    # {1: 'bool'}  — ONE key!
```
`1 == 1.0 == True` and all three hash to 1, so they are the *same key* — each assignment overwrote the value while keeping the **original** key object. This shows up for real when JSON gives you `1` and your code has `1.0`.

If you define `__eq__` on your own class, Python sets `__hash__` to `None` — your class becomes unhashable, because the default identity hash would violate the contract. You must define `__hash__` too (Lesson 06 covers this; `@dataclass(frozen=True)` does it for you).

### 2.5 `set` — a dict without values

Same hash table, no value slots. Sets give you O(1) membership and algebraic operations:

```python
a = {1, 2, 3}; b = {3, 4}
a | b        # union         {1,2,3,4}      a.union(b)
a & b        # intersection  {3}            a.intersection(b)
a - b        # difference    {1,2}          a.difference(b)
a ^ b        # symmetric diff {1,2,4}       a.symmetric_difference(b)
a <= b       # subset        False          a.issubset(b)
a.isdisjoint(b)   # False
```

Complexity: `&` is O(min(len(a), len(b))) — CPython iterates the smaller set. `|` and `-` are O(len(a)) / O(len(a)+len(b)). This is why set algebra beats nested loops so decisively.

The operator forms require both operands be sets; the method forms accept **any iterable** — `a.union([1,2])` works, `a | [1,2]` raises. Useful, and a nice interview detail.

`frozenset` is the immutable, hashable variant — the only way to use a set as a dict key or put sets in a set.

### 2.6 `tuple` — not just "immutable list"

Tuples are a different *idea*: a list is a homogeneous sequence of unknown length ("all the users"), a tuple is a heterogeneous record of known shape ("one row: id, name, email"). That's why `list[str]` and `tuple[int, str, str]` are the idiomatic annotations.

Practical benefits: hashable (usable as dict keys — the standard composite-key idiom), slightly smaller and faster to construct, and safe to share/return without defensive copying.

```python
sales = {}
sales[("2026-07", "EU", "widget")] = 1200      # composite key — extremely common
```

**Immutability is shallow**: `t = ([1], 2); t[0].append(3)` works fine. The tuple's *pointers* are frozen, not the objects. Corollary: a tuple containing a list is **unhashable**.

The infamous one:
```python
t = ([1, 2], "x")
t[0] += [3]      # TypeError: 'tuple' object does not support item assignment
t[0]             # [1, 2, 3]  — ...AND YET IT WORKED
```
`+=` on a list calls `__iadd__` (mutates in place, succeeds) then tries `t[0] = result`, which fails. The mutation already happened. This is *the* classic "explain this output" interview puzzle.

### 2.7 The `collections` module

**`defaultdict`** — a factory for missing keys. Kills the `if k not in d: d[k] = []` boilerplate.
```python
from collections import defaultdict
groups = defaultdict(list)
for user in users:
    groups[user.dept].append(user.name)      # no key check needed
```
Trap: *reading* a missing key **creates** it. `groups["nonexistent"]` returns `[]` and silently inserts it. Use `.get()` or convert to `dict()` when done if that matters.

**`Counter`** — a multiset.
```python
from collections import Counter
c = Counter("mississippi")
c.most_common(2)          # [('i', 4), ('s', 4)]
c["z"]                    # 0 — missing keys return 0, do NOT raise or insert
Counter("aab") + Counter("bc")     # Counter({'a': 2, 'b': 2, 'c': 1})
Counter("aab") - Counter("ab")     # Counter({'a': 1})  — negatives DROPPED
c.total()                 # 11 (3.10+)
```
Note `-` drops non-positive counts but `subtract()` keeps them. `most_common()` sorts, so it's O(n log n); `most_common(k)` uses `heapq.nlargest` — O(n log k).

**`deque`** — doubly-linked list of blocks. **O(1) at both ends.**
```python
from collections import deque
d = deque(maxlen=1000)               # bounded: appending past maxlen drops from the other end
d.appendleft(x); d.popleft()         # O(1) — the whole point
d.rotate(1)
```
`maxlen` gives you a **ring buffer in one argument** — the idiomatic "last N log lines" / sliding window. Trade-off: `d[i]` in the middle is **O(n)**, so a deque is not a list replacement, it's a queue.

**`namedtuple`** — a tuple with field names, zero extra memory.
```python
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(1, 2)
p.x, p[0]                 # 1, 1 — indexable AND attribute-accessible
p._replace(x=10)          # Point(x=10, y=2) — returns a NEW tuple
p._asdict()               # {'x': 1, 'y': 2}
```
Same memory as a plain tuple (no `__dict__`). Modern alternative: `typing.NamedTuple` (annotated, better tooling) or a frozen dataclass (Lesson 07) if you want methods and defaults.

**`OrderedDict`** — still useful post-3.7 for three things: `move_to_end()`, `popitem(last=False)`, and **order-sensitive equality** (`OrderedDict(a=1,b=2) != OrderedDict(b=2,a=1)`, but the plain dicts compare equal). It's the natural LRU backbone.

**`ChainMap`** — layered lookup without merging.
```python
from collections import ChainMap
config = ChainMap(cli_args, env_vars, defaults)   # first match wins, no copying
```
Layers stay live — updating `defaults` is visible immediately, and writes go to the **first** map only. Perfect for config precedence.

### 2.8 When to leave the builtins

- **`heapq`** — a priority queue on a plain list. `heappush`/`heappop` O(log n), `nlargest(k, it)` O(n log k). Use for top-k, merge-k-sorted (`heapq.merge`), schedulers.
- **`bisect`** — binary search on a *sorted* list. `bisect_left` O(log n), but `insort` is O(n) because of the shift. Great for read-heavy sorted lookups (rate tables, time buckets); bad for write-heavy — then use a real balanced tree (`sortedcontainers`).
- **`array.array`** — homogeneous C-typed array. A million ints: `list` ≈ 8 MB of pointers **plus** ~28 bytes/object ≈ 36 MB; `array('q')` = 8 MB flat.
- **NumPy** — when you want vectorised math, not a container.

---

## 3. Official documentation quotes

> "Dictionaries preserve insertion order. Note that updating a key does not affect the order. Keys added after deletion are inserted at the end."
> — *Python tutorial*, [5.5. Dictionaries](https://docs.python.org/3/tutorial/datastructures.html#dictionaries)

> "An object is hashable if it has a hash value which never changes during its lifetime (it needs a `__hash__()` method), and can be compared to other objects (it needs an `__eq__()` method). Hashable objects which compare equal must have the same hash value. … Objects which are instances of user-defined classes are hashable by default; they all compare unequal (except with themselves), and their hash value is derived from their `id()`."
> — *Python Glossary*, [hashable](https://docs.python.org/3/glossary.html#term-hashable)

> "Deques support thread-safe, memory efficient appends and pops from either side of the deque with approximately the same O(1) performance in either direction. Though `list` objects support similar operations, they are optimized for fast fixed-length operations and incur O(n) memory movement costs for `pop(0)` and `insert(0, v)` operations."
> — *Python Standard Library*, [`collections.deque`](https://docs.python.org/3/library/collections.html#collections.deque)

> "If `default_factory` is not `None`, it is called without arguments to provide a default value for the given key, this value is inserted in the dictionary for the key, and returned."
> — *Python Standard Library*, [`collections.defaultdict`](https://docs.python.org/3/library/collections.html#collections.defaultdict)

> "Tuples are immutable, and usually contain a heterogeneous sequence of elements … Lists are mutable, and their elements are usually homogeneous and are accessed by iterating over the list."
> — *Python tutorial*, [5.3. Tuples and Sequences](https://docs.python.org/3/tutorial/datastructures.html#tuples-and-sequences)

---

## 4. Real-world examples

### 4.1 The O(n²) bug and its one-character fix

```python
import random, time

haystack = [random.randint(0, 10_000_000) for _ in range(50_000)]
needles  = [random.randint(0, 10_000_000) for _ in range(50_000)]

start = time.perf_counter()
found_list = [n for n in needles if n in haystack]          # O(n*m)
list_time = time.perf_counter() - start

hay_set = set(haystack)                                     # O(n) once
start = time.perf_counter()
found_set = [n for n in needles if n in hay_set]            # O(n)
set_time = time.perf_counter() - start

assert sorted(found_list) == sorted(found_set)
print(f"list: {list_time:6.3f}s   set: {set_time:6.3f}s   speedup: {list_time/set_time:.0f}x")
```
```
list: 11.482s   set:  0.004s   speedup: 2871x
```

Same result, ~3000× apart, and the gap **grows linearly with data size**. This is the most valuable single lesson in the file.

### 4.2 Grouping and counting — the data-engineering bread and butter

```python
from collections import defaultdict, Counter

rows = [
    {"user": "ann", "action": "login",  "dept": "eng"},
    {"user": "bob", "action": "logout", "dept": "eng"},
    {"user": "ann", "action": "login",  "dept": "eng"},
    {"user": "cat", "action": "login",  "dept": "sales"},
]

by_dept = defaultdict(list)
for r in rows:
    by_dept[r["dept"]].append(r["user"])
print(dict(by_dept))

actions = Counter(r["action"] for r in rows)     # generator — no intermediate list
print(actions, "| top:", actions.most_common(1))

# two-level grouping
nested = defaultdict(lambda: defaultdict(int))
for r in rows:
    nested[r["dept"]][r["action"]] += 1
print({k: dict(v) for k, v in nested.items()})
```
```
{'eng': ['ann', 'bob', 'ann'], 'sales': ['cat']}
Counter({'login': 3, 'logout': 1}) | top: [('login', 3)]
{'eng': {'login': 2, 'logout': 1}, 'sales': {'login': 1}}
```

Note `defaultdict(lambda: defaultdict(int))` — the factory must be a *callable*, so nesting needs a lambda. This is exactly a `GROUP BY dept, action` and worth holding next to the SQL track.

### 4.3 `deque` as a bounded log tail

```python
from collections import deque

last_errors = deque(maxlen=5)                # ring buffer, fixed memory forever
for i in range(100):
    last_errors.append(f"error #{i}")
print(list(last_errors))
```
```
['error #95', 'error #96', 'error #97', 'error #98', 'error #99']
```
Constant memory regardless of input volume — the standard shape for in-process "recent events" endpoints and health dashboards.

### 4.4 A hash join, by hand

```python
users  = [{"id": 1, "name": "Ann"}, {"id": 2, "name": "Bob"}]
orders = [{"user_id": 1, "total": 50}, {"user_id": 1, "total": 20}, {"user_id": 3, "total": 99}]

index = {u["id"]: u for u in users}                  # BUILD phase — O(n)
for o in orders:                                      # PROBE phase — O(m)
    user = index.get(o["user_id"])
    print(f"{user['name'] if user else '<orphan>'}: {o['total']}")
```
```
Ann: 50
Ann: 20
<orphan>: 99
```
This *is* what a database's hash join does (SQL Lesson 03/09): build a hash table on the smaller side, stream the larger side through it. The `<orphan>` row is why `LEFT JOIN` exists. Understanding the symmetry between Python dicts and SQL joins is a genuine senior signal.

### 4.5 Measuring the shape cost

```python
import sys
from collections import namedtuple
from dataclasses import dataclass

Row = namedtuple("Row", "a b c")
@dataclass
class DRow:
    a: int; b: int; c: int
@dataclass
class SRow:
    __slots__ = ("a", "b", "c")
    a: int; b: int; c: int

print("dict      ", sys.getsizeof({"a":1,"b":2,"c":3}))
print("tuple     ", sys.getsizeof((1,2,3)))
print("namedtuple", sys.getsizeof(Row(1,2,3)))
print("dataclass ", sys.getsizeof(DRow(1,2,3)) + sys.getsizeof(DRow(1,2,3).__dict__))
print("slots     ", sys.getsizeof(SRow(1,2,3)))
```
```
dict       184
tuple       64
namedtuple  64
dataclass  152
slots       56
```
At 10 million rows that's 1.8 GB vs 0.6 GB — the difference between fitting on the box and not. `sys.getsizeof` is shallow (it doesn't count the ints), which is fine here since all variants share the same int objects.

---

## 5. How This Is Used In Production

- **Startups:** `Counter` and `defaultdict` are the default analytics engine before anyone installs Pandas — "top 10 endpoints by error rate" is three lines. `set` deduplication guards against duplicate webhook deliveries. `deque(maxlen=N)` backs the "recent activity" panel every product ships in month two.
- **Enterprises:** container choice becomes a cost line. A batch job holding 50M records as a list-of-dicts needs ~9 GB; as a list of `__slots__` objects or an `array`/NumPy structure it's 1–2 GB, which changes the instance type and the bill. Enterprises also standardise on `frozenset` for permission/scope checks, because `required_scopes <= granted_scopes` is one O(k) operation instead of a loop, and it's obviously correct at review time.
- **Common architectures:** the **hash join** (§4.4) is the core of every in-memory join, in Python and in Postgres alike. Deduplication pipelines use a `set` of seen IDs (or a Bloom filter once the set exceeds RAM — that's the moment to reach for `pybloom`/Redis). Work queues are `deque` in-process, Redis lists / SQS across processes. Rate limiters are `deque` of timestamps (sliding window) or a counter with a TTL.
- **Scaling:** the progression is predictable and worth memorising — **list → set/dict** (fix the O(n²)); **dict-of-lists → `array`/NumPy** (fix the per-object overhead); **in-process set → Redis set / Bloom filter** (fix "doesn't fit in RAM"); **Python containers → Arrow/Parquet columnar** (fix "doesn't fit on the box"). Each step buys roughly an order of magnitude and costs some flexibility.
- **Monitoring:** `Counter` is the mental model for a Prometheus counter; `deque(maxlen=N)` is a crude histogram/reservoir. In-process metrics libraries (`prometheus_client`) are dicts of label-tuples → values — which is precisely why label cardinality explodes memory: each unique label combination is a new dict key that never expires. Teams have OOM'd Prometheus by putting a user ID in a label.
- **Security:** two real ones. **Hash-collision DoS** — pre-3.3, an attacker could POST thousands of colliding form keys and turn dict inserts into O(n²), pinning a CPU; PEP 456's SipHash with per-process randomisation fixed it, which is why `hash("abc")` differs between runs and why you must never persist a `hash()` value. **Unbounded containers** — any dict keyed by user input and never evicted is a memory-exhaustion vector; bound it or TTL it.
- **CI/CD:** `set` operations drive test selection ("changed files ∩ files each test touches"), dependency resolution, and `git diff --name-only | ...` tooling. Deterministic output matters: `PYTHONHASHSEED=0` or sorting before output, because set iteration order is *not* stable across runs and will produce flaky snapshot tests.
- **Common tools:** `collections`, `heapq`, `bisect`, `array` (stdlib); `sortedcontainers` (pure-Python sorted list/dict/set with O(log n) inserts — genuinely faster than `bisect.insort` at scale); `frozendict`; `pyrsistent` (immutable structures); NumPy/Pandas/Polars/PyArrow for columnar.
- **Real examples:** **Instagram** (the largest Django deployment) has published extensively on Python memory tuning — shared-memory and object-overhead work driven by exactly the arithmetic in §4.5. **Dropbox** and **Google** both push `__slots__`/`namedtuple` for high-cardinality records in their Python style guidance. **Cloudflare**, **GitHub**, and every CDN use set-membership structures (and their probabilistic cousins) for cache and blocklist lookups at request rate.

---

## 6. Advanced corner

**Why `dict` ordering was an accident.** The 3.6 compact-dict change (Raymond Hettinger, on an idea from PyPy) split the hash table into a small sparse index array and a dense entries array. Iteration walks `entries`, which is append-ordered — so ordering fell out for free. Guido initially declared it an implementation detail; it was made a language guarantee in 3.7 because the ecosystem had already come to depend on it. The senior nuance: **`dict` is ordered, but `dict` equality still ignores order** (`{"a":1,"b":2} == {"b":2,"a":1}` is `True`) — only `OrderedDict` compares order-sensitively. `**kwargs` and `__dict__` ordering are guaranteed downstream effects.

**Deletion leaves tombstones.** Open addressing can't just blank a slot — that would break probe chains for other keys. It writes a `DUMMY` marker. Consequences: a dict that has had many insert/delete cycles accumulates tombstones and degrades until a resize cleans them, and **a dict never shrinks on deletion**. A dict that peaked at 10M keys keeps its table until you rebuild it (`d = dict(d)`). Same for sets. This is a real long-running-process leak.

**Iteration order after deletion is not "insertion order" in the naive sense.** Deleting from the entries array leaves a hole; a later insert may reuse compacted space after a resize. The guarantee is "keys added after deletion are inserted at the end" — don't build logic on anything subtler.

**`list.sort()` is Timsort** — a hybrid merge/insertion sort exploiting existing runs. O(n log n) worst case, **O(n) on already-sorted or reverse-sorted input**, and **stable** (equal elements keep their relative order). Stability is what makes multi-key sorting work by sorting repeatedly, least-significant key first:
```python
rows.sort(key=lambda r: r["name"])          # secondary
rows.sort(key=lambda r: r["dept"])          # primary — ties keep name order
```
Timsort was invented for CPython by Tim Peters and has since been adopted by Java, Android, V8, and Rust. `sorted()` returns a new list (O(n) extra memory); `.sort()` is in-place. **`key=` is called once per element (O(n) calls); `cmp` was removed precisely because it was called O(n log n) times.** The decorate-sort-undecorate pattern is now just `key=`.

**Small-int and string interning.** `-5..256` are singletons, and identifier-like strings are interned at compile time, so `a is b` can be `True` for small ints and *some* strings. **Never use `is` for value comparison** — this is why. `256 is 256` → `True`; `257 is 257` → depends on whether they're in the same code object. It's an implementation detail with no guarantees.

**`sys.getsizeof` lies (usefully).** It's shallow: `getsizeof([1,2,3])` counts the array of 3 pointers, not the ints. For real numbers use `tracemalloc`, `pympler.asizeof`, or measure RSS. The interview-worthy figure: a bare `object()` is 16 bytes, an `int` is 28, an empty `dict` is 64, an empty `list` is 56 — so a "small" record modelled as a dict costs ~200 bytes before any data.

**`__slots__` and why it works.** Declaring `__slots__` removes the per-instance `__dict__` and stores attributes in a fixed C array — saving ~50–60% for small objects and making attribute access marginally faster. Costs: no dynamic attributes, no `__weakref__` unless you add it, and multiple inheritance with slots is fiddly. For a class with millions of instances, it's the single highest-leverage memory change available (§4.5).

**What most people get wrong at this level:**
- Using `list` where `set` belongs, and never noticing because their test fixture has 10 rows.
- Believing `dict` lookups are "always O(1)" — average, not worst case, and the constant matters (hash + probe + `__eq__`), which is why `bisect` on a sorted list can beat a dict for small n.
- Thinking `tuple` means "deeply immutable." It doesn't (§2.6).
- Assuming `defaultdict` is a drop-in `dict` — reads mutate it, which corrupts membership tests and iteration-while-reading.
- Reaching for `OrderedDict` "because dicts aren't ordered" (outdated since 3.7) or dismissing it entirely (`move_to_end` and order-sensitive `==` are still unique).
- Not knowing that Python's sort is stable, and therefore reinventing multi-key sorting with a giant `cmp_to_key`.

---

## 7. Best practices

- **`in` on a `list` inside a loop is a bug.** Build a `set` once, outside. If the list is small and the loop runs once, fine — but write the `set` anyway; it documents intent.
- Choose by operation, not by habit: *lookup by key* → dict; *membership/dedup* → set; *ordered sequence you index* → list; *both ends* → deque; *fixed record* → tuple/NamedTuple/frozen dataclass; *top-k* → heapq; *sorted read-heavy* → bisect.
- Use `defaultdict`/`Counter` instead of `if k in d` boilerplate — then `dict(dd)` before returning it from a public API, so callers don't inherit the auto-insert behaviour.
- Prefer `d.get(k, default)` for reads and `d.setdefault(k, [])` only when you genuinely want the insert. `setdefault` **always evaluates its default** — `d.setdefault(k, expensive())` calls `expensive()` every time.
- Use `frozenset` for constant sets (module-level `VALID_STATES = frozenset({...})`) — hashable, immutable, and signals intent.
- Use tuple keys for composite lookups (`sales[(month, region)]`) rather than string concatenation (`f"{month}:{region}"`) — no delimiter-injection bugs, no parsing back.
- Sort with `key=`, never `cmp_to_key`, and lean on stability for multi-key sorts.
- Bound anything keyed by user input: `deque(maxlen=)`, `lru_cache(maxsize=)`, TTL caches. Unbounded is a memory leak with a delay fuse.
- Add `__slots__` to any class you'll instantiate more than ~100k times.
- Iterate `d.items()` when you need both; `d.keys()`/`.values()`/`.items()` are **views** — lazy and live, so `list(d)` if you'll mutate while iterating.

## 8. Common mistakes & gotchas

**Mutating while iterating**
```python
d = {"a": 1, "b": 2}
for k in d:
    if d[k] == 1:
        del d[k]          # RuntimeError: dictionary changed size during iteration
for k in list(d):         # snapshot the keys first
    ...
```

**`defaultdict` reads insert**
```python
dd = defaultdict(list)
if dd["missing"]:         # creates dd["missing"] = []
    ...
len(dd)                   # 1 — the check itself grew the dict
```

**`1 == 1.0 == True` collapse into one key** (§2.4).

**Shallow copy surprises**
```python
grid = [[0] * 3] * 3      # THREE REFERENCES TO ONE LIST
grid[0][0] = 9
grid                      # [[9,0,0],[9,0,0],[9,0,0]]
grid = [[0] * 3 for _ in range(3)]      # correct
```
`* 3` copies the pointer, not the list. Same reason `a = b = []` shares.

**The tuple `+=` puzzle** (§2.6) — mutates *and* raises.

**Set iteration order is not stable across runs** — `PYTHONHASHSEED` randomises string hashing. `sorted(my_set)` before printing/asserting or your tests flake in CI only.

**`sorted()` on mixed types** — `sorted([1, "a"])` raises `TypeError` (no Python 2-style arbitrary ordering).

**Dicts never shrink** — deleting 9M of 10M keys frees the values but keeps the table.

**`list.remove(x)` is O(n) and removes only the first match**; `list.index(x)` raises `ValueError` if missing (unlike `.get`).

## 9. Where AI helps (and where it hurts)

- **Helps:** picking the right `collections` type for a described problem (it's genuinely good at "I need to count/group/window..."); generating `Counter`/`defaultdict` one-liners; explaining `heapq`'s API, which is unintuitive (min-heap only, invert for max-heap, `heapify` is O(n) not O(n log n)); writing the benchmark harness for §4.1.
- **Hurts:** AI reproduces the O(n²) `in list` pattern constantly, because it's the most common pattern in its training data — it optimises for looking natural, not for complexity. It also confidently claims dicts are "unordered" (pre-3.7 knowledge that survives in the corpus), sometimes suggests `OrderedDict` where a plain dict is fine, and will happily hand you `[[0]*3]*3`. Ask it explicitly: *"what's the time complexity of this and where does it degrade?"* — it usually answers correctly once asked, but rarely volunteers it.

## 10. Learn independently

**Books (specific chapters):**
- *Fluent Python*, 2nd ed. (Ramalho) — **Ch. 2 "An Array of Sequences"** (list vs tuple, `array`, `deque`, and the `+=` puzzle), **Ch. 3 "Dictionaries and Sets"** (the compact-dict layout, hash internals, `dict` views — the best treatment in any book), **Ch. 11 "A Pythonic Object"** (`__slots__`, `__hash__`/`__eq__`). Ch. 3 is the single most valuable chapter for this lesson.
- *High Performance Python*, 2nd ed. (Gorelick & Ozsvald) — **Ch. 3 "Lists and Tuples"**, **Ch. 4 "Dictionaries and Sets"** (with the probing/collision walkthrough and memory measurements), **Ch. 11 "Using Less RAM"**. Read Ch. 4's hash-table section alongside Fluent Python's.
- *Effective Python*, 2nd ed. (Slatkin) — **Item 16** (`get` over `in`/`KeyError`), **Item 17** (`defaultdict` over `setdefault`), **Item 18** (`__missing__`), **Item 71–72** (`deque` for producer-consumer, `bisect` for searching sorted sequences), **Item 73** (`heapq` for priority queues).
- *Python Cookbook*, 3rd ed. (Beazley & Jones) — **Ch. 1 "Data Structures and Algorithms"** — 20 recipes that are effectively this lesson's homework with solutions.
- *CPython Internals* (Shaw) — the objects chapters, for the C-level view of `listobject.c` and `dictobject.c`.

**Primary docs:**
- [Data structures tutorial](https://docs.python.org/3/tutorial/datastructures.html)
- [`collections`](https://docs.python.org/3/library/collections.html) — read it end to end once; it's short.
- [`heapq`](https://docs.python.org/3/library/heapq.html) and [`bisect`](https://docs.python.org/3/library/bisect.html)
- [TimeComplexity wiki page](https://wiki.python.org/moin/TimeComplexity) — the canonical table. **Print this.**
- [PEP 468](https://peps.python.org/pep-0468/) (ordered `**kwargs`), [PEP 456](https://peps.python.org/pep-0456/) (SipHash / hash randomisation).
- [`dictobject.c` source](https://github.com/python/cpython/blob/main/Objects/dictobject.c) — the header comment is a genuinely readable design document.

**High-quality links:**
- Raymond Hettinger, *"Modern Dictionaries"* (PyCon 2017 talk) — the compact-dict design from its author. The definitive source.
- Brandon Rhodes, *"The Mighty Dictionary"* (PyCon 2010) — still the clearest visual explanation of open addressing.
- [`sortedcontainers` docs](https://grantjenks.com/docs/sortedcontainers/) — including its performance-comparison page, which is an education in itself.

## 11. Interview preparation

**Q1. What's the time complexity of `x in list` vs `x in set`?**
> O(n) vs O(1) average. A list scans element by element calling `__eq__`; a set hashes `x` once and probes a bucket. The set's O(1) is *average* — worst case is O(n) if all keys collide, which is why Python randomises string hashing to stop attackers engineering collisions. The practical rule: if you test membership more than a couple of times, build a set first; the O(n) build cost is amortised immediately.

**Q2. Why can't you use a list as a dict key?**
> Dict keys must be hashable, meaning a hash that never changes for the object's lifetime. Lists are mutable, so their hash would change as their contents changed — the key would then hash to a different bucket than the one it's stored in and become unreachable, silently corrupting the dict. Python prevents the entire bug class by setting `list.__hash__ = None`. Use a tuple (if the contents are themselves hashable) or a `frozenset`.

**Q3. Are dicts ordered? Since when, and why?**
> Yes, insertion-ordered — an implementation detail in CPython 3.6 and a language guarantee from 3.7. It was a side effect of the compact-dict memory optimisation: the table was split into a sparse array of indices and a *dense* array of entries appended in insertion order, cutting memory ~20–25%; iteration walks the dense array, so order fell out for free. Two nuances: dict *equality* still ignores order (only `OrderedDict` compares order-sensitively), and sets are still unordered because they got no equivalent rework.

**Q4. When would you use `deque` over `list`?**
> When you need O(1) operations at the **front**. `list.pop(0)`/`insert(0, x)` are O(n) because every element shifts; `deque` is a doubly-linked list of blocks, so both ends are O(1). Queues, BFS, sliding windows, and `deque(maxlen=N)` ring buffers. The trade-off: random access `d[i]` in the middle is O(n), so a deque is not a general list replacement.

**Q5. `tuple` vs `list` — beyond mutability?**
> Semantics first: a tuple is a **record** of heterogeneous fields with fixed meaning (`(id, name, email)`), a list is a **sequence** of homogeneous items of unknown length. That's why the annotations differ: `tuple[int, str, str]` vs `list[str]`. Practically, tuples are hashable (usable as dict keys — the composite-key idiom), marginally smaller and faster to construct, and safe to return without defensive copying. The catch: immutability is **shallow** — a tuple containing a list can still be mutated through it, and is unhashable.

**Q6. Explain this:** `t = ([1,2], 3); t[0] += [4]` **raises `TypeError` but `t[0]` becomes `[1,2,4]`.**
> `+=` on a list compiles to: evaluate `t[0]`, call its `__iadd__` (which extends the list **in place** and returns it), then **store the result back** via `t[0] = ...`. Step two succeeds — the mutation is already done — and step three raises because tuples don't support item assignment. So you get the exception *and* the mutation. The lesson: don't put mutable objects in tuples, and know that `+=` is not atomic.

**Q7. (System design) Deduplicate a 500 GB event stream by event ID. Walk me through it.**
> Start with the naive shape and scale it. A Python `set` of 500M UUID strings is roughly 100+ bytes each with table overhead — ~50–75 GB. Doesn't fit. Options, in order of what I'd try: **(1) Partition** — hash the ID modulo N and process each partition independently, so each set fits in RAM; this is exactly how MapReduce shuffles and it's usually the right answer because it's exact. **(2) External sort** — sort by ID (Timsort's merge on chunks, or just `sort -u`) and dedupe adjacent runs; O(n log n) I/O but constant memory. **(3) Bloom filter** — ~1.2 GB at 1% FPR for 500M items, but it gives false *positives*, so it can only be a cheap pre-filter in front of an exact check unless dropping some legitimate events is acceptable. **(4) Redis/RocksDB set** — exact, spills to disk, but adds a network hop per event. I'd default to (1) if I control the pipeline, (3)+(4) if it's a streaming system where I need per-event decisions with bounded latency. The key trade-off is exactness vs memory, and you have to make the business tell you whether a 1% false-dedup rate is acceptable — usually it isn't for billing, usually it is for analytics.

**Q8. (Trade-off) A million records — list of dicts, list of tuples, `__slots__` class, or NumPy?**
> Roughly: dicts ≈ 184 B each (~184 MB + the values) and give you flexible schema and JSON-shaped code; namedtuples ≈ 64 B and give you names for free but no methods and no mutation; `__slots__` classes ≈ 56 B plus methods and type-checkability; NumPy structured arrays are ~the raw byte size with vectorised operations but a fixed schema and awkward strings. I'd start with dicts for clarity, move to `__slots__`/`NamedTuple` when memory or attribute-typo bugs bite, and only go NumPy/Arrow when I'm doing real numeric work or the dataset is genuinely large — because at that point I want Parquet and columnar I/O anyway, not just a smaller in-memory object. The meta-point: measure with `tracemalloc` before optimising; the object overhead only matters above ~1M instances.

**Q9. What's a hash collision and how does CPython handle it?**
> Two distinct keys hashing to the same table slot. CPython uses **open addressing**, not chaining: on collision it probes another slot in the same table using a perturbation sequence that mixes in the *upper* bits of the hash, so clusters spread out rather than piling up linearly. The table resizes when 2/3 full because open addressing degrades sharply near capacity. Deletion writes a tombstone rather than blanking (blanking would break probe chains for other keys), which is why dicts never shrink and why heavy insert/delete churn degrades until a resize. Chaining (Java's `HashMap`) is the alternative — better under high load factors, worse cache locality.

**Q10. Why is `sorted()` stable, and why do you care?**
> Timsort preserves the relative order of elements that compare equal. You care because it makes multi-key sorting trivial: sort by the least significant key first, then the most significant, and the earlier order survives within ties. It also means sorting is idempotent and diffs are stable, which matters for reproducible output. Timsort is additionally adaptive — it detects existing sorted runs, so nearly-sorted data (very common in real pipelines: log lines are almost always near-chronological) sorts in near-O(n).

**Q11. What does `__slots__` do and when would you skip it?**
> It replaces the per-instance `__dict__` with a fixed array of descriptor-backed slots, cutting small-object memory by roughly half and speeding up attribute access slightly. Skip it when you have few instances (the complexity isn't worth it), when you need dynamic attributes or `setattr` of arbitrary names, when you need `__weakref__` (you must add it explicitly), or when multiple inheritance makes the slot layout conflict. It's an optimisation for the million-instance case, not a default.

**Q12. (Senior) Your service's memory grows steadily and never returns after load spikes, but there's no leak in the usual sense. What are your suspects?**
> First: containers that never shrink. A `dict` or `set` that peaked during a spike keeps its table allocated after deletion, and a `list` keeps its over-allocated array — the *values* are freed but the table isn't, so RSS stays high. Rebuild (`d = dict(d)`) or bound them. Second: unbounded caches — `@lru_cache` without `maxsize`, or a dict keyed on something user-controlled (session ID, URL, metric label). Third: `lru_cache` on a method, pinning every instance ever passed. Fourth: fragmentation — CPython returns freed arenas to the OS only when fully empty, so a long-lived process can hold pages it isn't using. I'd confirm with `tracemalloc` snapshot diffs across the spike and `gc.get_objects()` counts by type; if tracemalloc shows flat Python allocations but RSS is high, it's arena fragmentation or a C extension, and the answer is a worker recycle policy (`--max-requests` in Gunicorn) rather than a code fix.

## 12. Homework

> Work in `python/solutions/lesson-05/`. Where a task asks for measurements, include the raw numbers in `RESULTS.md` — from *your* machine, not from this lesson.

**Easy**

1. **Complexity by measurement.** Write `bench_membership.py` that, for n in `[100, 1_000, 10_000, 100_000]`, times 1,000 membership tests against a `list` and against a `set`. Print a table of n, list_ms, set_ms, ratio.
   *Acceptance:* the ratio grows roughly linearly with n for the list and stays flat for the set; you can state from your own numbers at what n the set build cost pays for itself.

2. **Container chooser.** For each scenario, name the structure and justify in one sentence: (a) last 100 log lines; (b) "have I seen this event ID?"; (c) count words in a document; (d) group orders by customer; (e) top 5 slowest queries out of 10M; (f) config with CLI > env > defaults precedence; (g) a cache key made of `(user_id, endpoint, date)`; (h) find the time bucket a timestamp falls in, from a sorted list of 10k boundaries.
   *Acceptance:* a `CHOICES.md` with structure + complexity + one-line rationale for each. (Answers span `deque`, `set`, `Counter`, `defaultdict`, `heapq`, `ChainMap`, `tuple`, `bisect`.)

3. **Gotcha gallery.** Reproduce, in one script with printed output and comments explaining *why*: the `[[0]*3]*3` aliasing bug; the `{1: 'int', 1.0: 'float', True: 'bool'}` collapse; the `defaultdict` read-inserts behaviour; the tuple `+=` puzzle; a `RuntimeError` from mutating a dict while iterating.
   *Acceptance:* five demonstrations, each with a one-paragraph explanation and a corrected version.

**Medium**

4. **Log analyser.** Given a generated Apache-style access log of 200k lines, produce with `collections` only (no Pandas): total requests; top 10 IPs; status-code distribution; top 5 URLs by 5xx rate (min 20 requests, to avoid tiny-denominator noise); the busiest minute; and the last 20 5xx lines. Stream the file lazily.
   *Acceptance:* runs in under 2 seconds on 200k lines; peak memory (via `tracemalloc`) does not scale with file size except for the `Counter` cardinality; a generator for the last-20 that uses `deque(maxlen=20)`.

5. **Hash join with all four join types.** Implement `join(left, right, on, how)` for `how` in `{"inner", "left", "right", "outer"}` over lists of dicts, using a dict index on the smaller side. Handle duplicate keys on both sides (a true cartesian product per key group) and missing values as `None`.
   *Acceptance:* pytest comparing your results against hand-written expected output for a case with duplicates on both sides; a docstring stating the complexity and which side you index and why.

6. **LRU cache from scratch.** Implement `LRUCache(capacity)` with `get`/`put`, O(1) both, using `OrderedDict` + `move_to_end`. Then implement it a **second** time using a raw dict + a hand-built doubly-linked list, no `OrderedDict`. Benchmark both.
   *Acceptance:* pytest covering eviction order, updating an existing key refreshing recency, and capacity 0/1 edge cases; a benchmark showing both are O(1) (flat time as capacity grows).

7. **Sliding-window rate limiter.** Implement `RateLimiter(max_calls, per_seconds)` with an `allow(key) -> bool`, using a `defaultdict(deque)` of timestamps and evicting expired ones on each call. Add `cleanup()` to drop empty keys.
   *Acceptance:* tests with a monkeypatched clock (no `sleep`) proving the window slides correctly at boundaries; a written note on the memory risk of `defaultdict(deque)` keyed by user-supplied IDs, and what you'd do about it in production.

**Hard**

8. **Memory shootout.** Model a record with 6 fields (2 ints, 3 short strings, 1 float) **six** ways: dict, tuple, `namedtuple`, `typing.NamedTuple`, plain dataclass, `__slots__` dataclass. Create 1,000,000 of each (one at a time, in a subprocess, forcing `gc.collect()`) and measure real RSS with `psutil` plus `tracemalloc` peak.
   *Acceptance:* a table of measured bytes/record for all six, the ratio to the theoretical minimum, and a recommendation paragraph. Explain any divergence between `tracemalloc` and RSS.

9. **Prove dicts don't shrink.** Build a dict of 5M keys, measure RSS; delete 90% of the keys, `gc.collect()`, measure again; rebuild with `d = dict(d)`, measure again. Repeat for a `set` and a `list`. Then construct a realistic scenario (a cache keyed by session ID during a traffic spike) and show the sawtooth-that-never-comes-down.
   *Acceptance:* `RESULTS.md` with three RSS numbers per container type, an explanation referencing tombstones and over-allocation, and two production mitigations you'd actually deploy.

10. **`__hash__`/`__eq__` contract violation.** Write a class with a mutable field used in `__eq__` and `__hash__`. Put an instance in a set, mutate the field, then try to find it (`x in s`) — show it's lost, and that `s` now contains an item that `in` can't reach and iteration *can*. Then show what happens with two objects that are `==` but have different hashes. Fix with `@dataclass(frozen=True)`.
    *Acceptance:* a script demonstrating both violations with printed evidence, plus a written statement of the contract and why each violation breaks the hash table specifically.

**Stretch (senior)**

11. **Word-frequency at scale.** Generate a 2 GB text file. Count word frequencies four ways: (a) naive `dict` in one pass; (b) `Counter` in one pass; (c) chunked with partition-by-hash into N temp files then merge (map-reduce shape); (d) `sort | uniq -c` shelled out. Measure wall time and peak RSS for each.
    *Acceptance:* a comparison table, and a paragraph on why (c) is the only one that generalises past RAM and what it costs you. Bonus: implement (c) with `multiprocessing` and report the speedup vs. core count.

12. **Beat `bisect` with `sortedcontainers`.** Build a sorted structure of 1M timestamped events supporting: insert, "events between t1 and t2", and "nearest event to t". Implement with `bisect.insort` on a list, then with `sortedcontainers.SortedList`. Benchmark 100k inserts interleaved with 100k range queries.
    *Acceptance:* measured numbers showing where `insort`'s O(n) shift dominates; a recommendation with the crossover point from your data; and a note on what a database would use instead (and why a B-tree, not a sorted array).

## 13. Key takeaways

- **`list` = dynamic array of pointers** (O(1) append amortised via geometric over-allocation, **O(n) at the front**); **`dict`/`set` = compact open-addressed hash tables** (O(1) average, insertion-ordered since 3.7 as a *side effect* of a memory optimisation).
- `x in list` is O(n) and `x in set` is O(1) — this one line is the most common real Python performance bug in existence.
- Hashability = a stable hash + a consistent `__eq__`. Mutable ⇒ unhashable ⇒ not a key. `1 == 1.0 == True` are one dict key.
- Tuples are **records**, lists are **sequences** — and tuple immutability is **shallow**.
- `collections` earns its place: `defaultdict` (grouping), `Counter` (counting), `deque` (both ends + `maxlen` ring buffer), `NamedTuple` (cheap records), `ChainMap` (layered config), `OrderedDict` (`move_to_end`, order-sensitive `==`).
- Containers **never shrink** and iteration order of sets isn't stable across runs — both cause real production and CI bugs.
- The scaling ladder: list→set (fix complexity) → `__slots__`/`array` (fix per-object overhead) → Redis/Bloom (fix "exceeds RAM") → Arrow/Parquet (fix "exceeds the box").
