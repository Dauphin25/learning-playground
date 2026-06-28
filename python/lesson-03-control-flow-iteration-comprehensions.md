# Python Lesson 03 — Control Flow, Iteration & Comprehensions

| | |
|---|---|
| **Track** | Python |
| **Lesson** | 03 of 15 |
| **Topic** | Branching, looping, the iterator protocol, and comprehensions |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–02 |

### Learning objectives
1. Use `if`/`elif`/`else`, truthiness, and `match` correctly and idiomatically.
2. Master `for`/`while`, `break`/`continue`/`else`-on-loops, and `enumerate`/`zip`.
3. Explain the **iterator protocol** (`__iter__`/`__next__`) and what "iterable" means.
4. Write list/dict/set comprehensions and generator expressions, and know when *not* to.
5. Understand lazy evaluation and why it matters for memory.

---

## 1. Why this matters
Control flow and iteration are where Python code is either *Pythonic and readable* or *clunky and slow*. Comprehensions and the iterator protocol are everywhere in data engineering (streaming rows lazily so you don't load a 10 GB file into RAM) and are a frequent interview discriminator. This lesson turns "I can write a loop" into "I write idiomatic, memory-aware Python."

---

## 2. Theory

### 2.1 Truthiness
Every object is truthy or falsy. **Falsy:** `None`, `False`, `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `()`, and objects defining `__bool__`/`__len__` returning false/0. Everything else is truthy.

```python
items = []
if items:            # idiomatic: "if the list is non-empty"
    process(items)
# NOT: if len(items) > 0  (works, but noisier)
```

### 2.2 Conditionals
```python
if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
else:
    grade = "C"

# conditional expression (ternary)
label = "even" if n % 2 == 0 else "odd"
```

### 2.3 Structural pattern matching (`match`, Python 3.10+)
More than a switch — it destructures.
```python
def describe(point):
    match point:
        case (0, 0):
            return "origin"
        case (x, 0):
            return f"on x-axis at {x}"
        case (0, y):
            return f"on y-axis at {y}"
        case (x, y):
            return f"at {x}, {y}"
        case _:
            return "not a 2D point"
```

### 2.4 Loops
```python
for i, name in enumerate(names, start=1):   # index + value
    print(i, name)

for a, b in zip(list1, list2):              # parallel iteration
    ...

while not done:
    done = step()
```
- `break` exits the loop; `continue` skips to the next iteration.
- **`for/else`**: the `else` runs only if the loop finished *without* `break` — useful for search loops:
```python
for item in haystack:
    if item == needle:
        print("found"); break
else:
    print("not found")   # only if we never broke
```

### 2.5 The iterator protocol (the heart of Python iteration)
- An **iterable** is anything you can loop over — it implements `__iter__()` returning an iterator (lists, dicts, files, ranges).
- An **iterator** implements `__next__()`, returning the next value or raising `StopIteration`. It's also an iterable (returns itself from `__iter__`).

`for x in obj:` is roughly:
```python
it = iter(obj)            # calls obj.__iter__()
while True:
    try:
        x = next(it)      # calls it.__next__()
    except StopIteration:
        break
    ...                   # loop body
```

A custom iterator:
```python
class Countdown:
    def __init__(self, n): self.n = n
    def __iter__(self): return self
    def __next__(self):
        if self.n <= 0:
            raise StopIteration
        self.n -= 1
        return self.n + 1

list(Countdown(3))   # [3, 2, 1]
```

### 2.6 Comprehensions
Concise, readable transformations:
```python
squares   = [x*x for x in range(10)]                 # list
evens     = [x for x in range(10) if x % 2 == 0]     # with filter
pairs     = [(x, y) for x in "ab" for y in (1, 2)]   # nested
by_len    = {w: len(w) for w in words}               # dict comprehension
unique    = {c for c in "mississippi"}               # set comprehension
```
Comprehensions are usually faster and clearer than the equivalent `for`+`append` loop, but **don't nest them three deep** — readability wins.

### 2.7 Generator expressions & laziness
Swap `[]` for `()` to get a **generator** — values are produced one at a time, lazily, using almost no memory:
```python
total = sum(x*x for x in range(1_000_000))   # no million-element list built
```
This is the difference between streaming and loading-everything — central to data work.

---

## 3. Official documentation quotes

> "By default, the `case` blocks are tried sequentially, and the first one that matches the subject is executed. … Patterns can also be used to bind variables."
> — *Python tutorial*, [match Statements](https://docs.python.org/3/tutorial/controlflow.html#match-statements)

> "Loop statements may have an `else` clause; it is executed when the loop terminates through exhaustion of the iterable (with `for`) or when the condition becomes false (with `while`), but not when the loop is terminated by a `break` statement."
> — *Python tutorial*, [break and continue Statements, and else Clauses on Loops](https://docs.python.org/3/tutorial/controlflow.html#break-and-continue-statements-and-else-clauses-on-loops)

> "Container objects can be iterated using a `for` statement … Behind the scenes, the `for` statement calls `iter()` on the container object. The function returns an iterator object that defines the method `__next__()`."
> — *Python tutorial*, [Iterators](https://docs.python.org/3/tutorial/classes.html#iterators)

> "List comprehensions provide a concise way to create lists. … A list comprehension consists of brackets containing an expression followed by a `for` clause, then zero or more `for` or `if` clauses."
> — *Python tutorial*, [List Comprehensions](https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions)

---

## 4. Real-world examples

### 4.1 Stream a large file lazily (don't load it all)
```python
def count_errors(path):
    count = 0
    with open(path) as f:                 # f is itself an iterator over lines
        for line in f:                    # one line at a time
            if "ERROR" in line:
                count += 1
    return count
```

### 4.2 Transform + filter in one comprehension
```python
records = [{"name": "  Ann ", "age": "30"}, {"name": "bob", "age": "x"}]
clean = [
    {"name": r["name"].strip().title(), "age": int(r["age"])}
    for r in records
    if r["age"].isdigit()
]
# [{'name': 'Ann', 'age': 30}]
```

### 4.3 `enumerate` + `zip` instead of index juggling
```python
headers = ["id", "name", "email"]
row     = [1, "Ann", "ann@x.com"]
record  = {h: v for h, v in zip(headers, row)}   # csv-row → dict
```

---

## 6. How This Is Used In Production
- **Startups:** comprehensions and generators are the day-to-day idiom for data munging — parsing API responses, reshaping records, filtering logs. Lazy file iteration (`for line in open(...)`) is the default way to process files larger than RAM.
- **Enterprises:** data pipelines stream records through chained generators so a single worker can process datasets far bigger than memory; this is the conceptual basis of tools like **Apache Beam**, **PySpark** transformations, and Python ETL frameworks. Pattern matching is increasingly used to dispatch on structured events/messages.
- **Common architectures:** producer → (generator) transform → consumer, where each stage `yield`s. Kafka/Kinesis consumers loop "forever" (`while True: for msg in consumer:`) processing records as they arrive — the iterator protocol applied to an infinite stream.
- **Scaling:** generators give **constant memory** regardless of input size — the key to scaling batch jobs vertically before you need a cluster. Comprehensions vectorize tight loops in pure Python; for heavy numeric work you then drop to NumPy/Pandas (which iterate in C).
- **Monitoring/Logging:** log processing is iteration over line streams; `grep`-like filters in Python are generator pipelines. Counters over streams power metrics.
- **Real examples:** **Spotify**, **Netflix**, and countless data teams build Python ETL/orchestration (often on **Apache Airflow**, itself Python) where lazy iteration keeps memory bounded. **Dropbox** and **YouTube** (early) processed enormous request/log volumes with Python generators feeding aggregation.

---

## 7. Best practices
- Prefer iterating the object directly (`for x in items`) over `for i in range(len(items))`.
- Use truthiness (`if items:`) over `len(items) > 0` / `== []`.
- Use `enumerate`/`zip` instead of manual index arithmetic.
- Reach for a **generator expression** when you only consume the result once or it's large.
- Keep comprehensions to one (maybe two) `for`/`if` clauses; beyond that, write a loop or a generator function.
- Use `match` when you're branching on the *shape/structure* of data, not for simple equality chains.

## 8. Common mistakes & gotchas
- **Mutating a list while iterating it** — skips items or errors. Iterate a copy (`for x in items[:]`) or build a new list.
- **Reusing an exhausted generator** — once consumed, it yields nothing more; recreate it.
- Forgetting that **`zip` stops at the shortest** iterable (use `itertools.zip_longest` if needed).
- Overusing comprehensions until they're unreadable; nesting 3+ levels.
- Building a full list with `[...]` when you only `sum()`/`any()`/`max()` it — use a generator expression to save memory.
- `range` is lazy in Python 3 — `list(range(n))` if you actually need the list.

## 9. Where AI helps (and where it hurts)
- **Helps:** rewriting a verbose loop as a clean comprehension (and back when it's gotten unreadable), generating `match` cases for a known data shape, explaining `for/else`.
- **Hurts:** AI loves to cram logic into one dense comprehension that's hard to debug; push back and ask for the readable version. It can also produce comprehensions with subtle scoping or ordering differences from your loop — test the output.

## 10. Learn independently
- [Control flow tutorial](https://docs.python.org/3/tutorial/controlflow.html) and [Data structures: comprehensions](https://docs.python.org/3/tutorial/datastructures.html).
- [`itertools`](https://docs.python.org/3/library/itertools.html) (preview for Lesson 12) — the iterator toolbox.
- *Fluent Python*, ch. on iterables, iterators, and generators.
- PEP 634–636 (pattern matching) for the `match` deep dive.

## 11. Interview preparation

**Q1. What's the difference between an iterable and an iterator?**
> An iterable implements `__iter__()` and can produce an iterator (lists, dicts, files). An iterator implements `__next__()` (and `__iter__` returning itself), yielding values until it raises `StopIteration`. `for` calls `iter()` on the iterable to get an iterator, then `next()` repeatedly.

**Q2. List comprehension vs generator expression — when use which?**
> A list comprehension `[...]` builds the whole list in memory; a generator expression `(...)` produces items lazily, one at a time. Use the generator when the data is large or consumed once (e.g., feeding `sum`/`any`); use the list when you need to index it, reuse it, or know its length.

**Q3. What does `for/else` do?**
> The `else` block runs only if the loop completed without hitting `break` — ideal for "search and report not-found" logic.

**Q4. How would you process a file too big to fit in memory?**
> Iterate it line by line (`for line in open(path)`), which is lazy, and chain generators for transforms so memory stays constant regardless of file size.

**Q5. What's wrong with modifying a list while iterating over it?**
> The iterator's internal index gets out of sync with the shrinking/growing list, causing skipped elements or runtime errors. Iterate over a copy or build a new list.

**Q6. `match` vs a chain of `if/elif` — what's the advantage?**
> `match` can destructure and bind from the structure of the data (tuples, classes, dicts), making structural dispatch clearer; plain `if/elif` is fine for simple scalar comparisons.

## 12. Homework
> Work in `python/solutions/lesson-03/`.

**Easy**
1. Given `nums = list(range(1, 21))`, use comprehensions to produce: (a) the squares of the even numbers, (b) a dict mapping each number to "fizz"/"buzz"/"fizzbuzz"/itself per FizzBuzz rules. No explicit `append`.

**Medium**
2. Write `wordcount.py` that reads a text file *lazily* and prints the 10 most common words (lowercased, punctuation stripped). Use a generator to stream lines and `collections.Counter`. It must not read the whole file into a single string.

**Hard**
3. Implement a class `Fibonacci` that is a proper iterator (`__iter__`/`__next__`) producing Fibonacci numbers up to a max value, then use it in a `for` loop and in a list comprehension. Separately, implement the same with a generator function and compare line counts. Add `for/else` logic that prints whether a given target value appeared in the sequence.

**Stretch:** Write a generator pipeline `parse → filter → transform` (three chained generator functions) that processes a CSV-like file lazily, and prove with `tracemalloc` that peak memory does not grow with file size (test on a small and a large generated file).

## 13. Key takeaways
- Truthiness drives idiomatic conditionals (`if items:`).
- `for/else`, `enumerate`, `zip` make loops clean; `match` destructures data.
- Iterable (`__iter__`) vs iterator (`__next__`) is the engine under every `for` loop.
- Comprehensions = readable transforms; generator expressions = lazy, constant-memory transforms.
- Laziness is how you process data bigger than RAM — the foundation of Python data engineering.
