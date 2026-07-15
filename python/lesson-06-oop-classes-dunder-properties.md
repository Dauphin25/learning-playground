# Python Lesson 06 — OOP I: Classes, Dunder Methods & Properties

| | |
|---|---|
| **Track** | Python |
| **Lesson** | 06 of 15 |
| **Topic** | Class construction, attribute lookup, the data model (dunders), properties & descriptors |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–05 (especially the object model in 02 and hashability in 05) |

### Learning objectives
1. Explain what `class` actually does at runtime and how attribute lookup really resolves (instance → class → MRO → `__getattr__`).
2. Implement the core dunders — `__init__`, `__repr__`, `__str__`, `__eq__`, `__hash__`, `__len__`, `__bool__`, `__iter__`, `__contains__`, `__call__` — and know which ones you get wrong most often.
3. Use `@property` to add computed and validated attributes *without* breaking your public API.
4. Explain the descriptor protocol and see that `property`, `classmethod`, `staticmethod`, `functions`, and `__slots__` are all descriptors.
5. Choose correctly between `@staticmethod`, `@classmethod`, and a module-level function.
6. Understand why Python has no real private attributes, what `_x` and `__x` actually mean, and what name mangling is for.

---

## 1. Why this matters

Python's OOP is unusual: there's no `private`, no interfaces in the Java sense, and — critically — **you never need getters and setters**, because `@property` lets you convert a plain attribute into a computed one *without changing a single call site*. This one fact demolishes the boilerplate that dominates Java/C# codebases and is the reason `obj.total` is idiomatic Python where `obj.getTotal()` is not.

The data model is where Python's real leverage lives. Implementing `__len__` and `__getitem__` makes your object work with `len()`, slicing, `for`, `in`, and `reversed()` — for free. This is what Ramalho calls the *Pythonic* payoff: you don't build an API, you **implement the protocol the language already speaks**.

In your target roles: SQLAlchemy models are descriptor-driven (`User.name == "x"` returns a SQL expression, not a bool — because `name` is a descriptor with a custom `__eq__` on the class attribute). Pydantic validates via descriptors and `__set_name__`. Every ORM, every serialisation library, every DI framework you'll use is built from this lesson's material. And "explain `__new__` vs `__init__`" / "what's a descriptor?" are the two questions that most reliably separate mid from senior in a Python interview.

---

## 2. Theory

### 2.1 `class` is an executable statement

Like `def`, `class` runs. It executes the class body in a fresh namespace, then calls the metaclass (normally `type`) with `(name, bases, namespace)` to build a class object, and binds the name.

```python
class Foo:
    x = 1              # this line RUNS, at import time
    print("executing") # so does this
    def m(self): ...

# is roughly:
Foo = type("Foo", (), {"x": 1, "m": <function m>})
```

Consequences:
- A class *is* an object (of type `type`). You can pass it, store it, and build it at runtime.
- Class-body code runs once at import. Expensive work there slows every import forever.
- **Class attributes are shared by all instances.** This is the mutable-class-attribute trap (§8), the exact sibling of the mutable-default trap from Lesson 04.

```python
class Bad:
    items = []              # ONE list, shared by every instance, forever
class Good:
    def __init__(self):
        self.items = []     # per instance
```

### 2.2 Attribute lookup — the real algorithm

`obj.x` is not a dict lookup. It's `type(obj).__getattribute__(obj, "x")`, which does roughly:

1. Look for `x` in `type(obj).__mro__` (the class and its bases). If found **and it's a data descriptor** (has `__set__` or `__delete__`) → call its `__get__`. **Data descriptors win over the instance dict.**
2. Look in `obj.__dict__`. If found → return it.
3. Back to the class-level result from step 1: if it's a **non-data descriptor** (only `__get__` — e.g. a plain function) → call `__get__`. Otherwise return it as-is.
4. Nothing found → call `type(obj).__getattr__(obj, "x")` if defined, else raise `AttributeError`.

Two things follow that people get wrong constantly:
- **`__getattr__` is a fallback, not a hook.** It runs *only when normal lookup fails*. `__getattribute__` runs on **every** access — override it and you will break things and create infinite recursion.
- **Data descriptors (`property`) cannot be shadowed by an instance attribute.** That's precisely why `@property` is safe: `self.x = 1` inside a class with a `property x` calls the property's setter, it doesn't blow the property away.

```python
class Demo:
    cls_attr = "class"
    def __init__(self):
        self.inst_attr = "instance"
    def __getattr__(self, name):          # only called on FAILURE
        return f"<computed {name}>"

d = Demo()
d.inst_attr    # 'instance'   — step 2
d.cls_attr     # 'class'      — step 3
d.whatever     # '<computed whatever>' — step 4
d.__dict__     # {'inst_attr': 'instance'} — cls_attr is NOT here
```

### 2.3 `__init__` is not a constructor

`__new__` allocates; `__init__` initialises. `Foo(1)` calls `type.__call__(Foo, 1)`, which does `obj = Foo.__new__(Foo, 1)` then, **if `obj` is an instance of `Foo`**, `obj.__init__(1)`, then returns `obj`.

```python
class Foo:
    def __new__(cls, *args):
        print("__new__ — allocating")
        return super().__new__(cls)      # must return an instance, or __init__ is skipped
    def __init__(self, x):
        print("__init__ — initialising")
        self.x = x
```

You almost never need `__new__`. The legitimate reasons: subclassing an immutable builtin (`int`, `str`, `tuple` — their value is fixed at allocation, so `__init__` is too late), implementing singletons/caching (return an existing instance), and metaclass work. If `__new__` returns something that isn't an instance of `cls`, **`__init__` is never called** — a subtle bug generator.

### 2.4 The dunders that earn their keep

**`__repr__` — write this one always.** Unambiguous, for developers, ideally `eval`-able.
**`__str__`** — readable, for users. Falls back to `__repr__` if absent, so if you write only one, write `__repr__`.

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __repr__(self):
        return f"Point(x={self.x!r}, y={self.y!r})"      # note !r on the fields
    def __str__(self):
        return f"({self.x}, {self.y})"

p = Point(1, 2)
p                # Point(x=1, y=2)   <- repr, in the REPL
print(p)         # (1, 2)            <- str
f"{p}"           # (1, 2)            <- str
f"{p!r}"         # Point(x=1, y=2)   <- forced repr
[p]              # [Point(x=1, y=2)] <- containers ALWAYS use repr of elements
```

That last line is why `__repr__` matters more than `__str__` in practice: debugging a list of objects with no `__repr__` gives you `[<__main__.Point object at 0x7f3a>]` — worthless. **Log lines and tracebacks use `repr`.** Missing `__repr__` costs you real debugging time in production.

**`__eq__` and `__hash__` — always together.**

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented          # NOT False — let Python try the reflected op
        return (self.x, self.y) == (other.x, other.y)
    def __hash__(self):
        return hash((self.x, self.y))      # hash the same fields you compare
```

Rules you must be able to recite:
- **Defining `__eq__` sets `__hash__ = None`** — your class becomes unhashable, unusable in sets/dict keys. This is deliberate: the inherited identity hash would violate the contract (`a == b` but `hash(a) != hash(b)`).
- Return **`NotImplemented`** (not `False`) for unknown types. Python then tries `other.__eq__(self)`, and only then falls back to identity. Returning `False` prematurely breaks interop with types that *could* compare with yours.
- Hash the **same fields** you compare, and those fields must be **immutable in practice** — mutating a field after inserting into a set loses the object (Lesson 05, Homework 10).
- `__ne__` is derived automatically from `__eq__` in Python 3. Don't write it.

**`__len__` and `__bool__`.** `bool(obj)` tries `__bool__`, then `__len__`, then defaults to `True`. So an object with `__len__` returning 0 is **falsy** — which is usually what you want for a container, but a nasty surprise if you added `__len__` to a non-container.

**`__iter__` / `__contains__` / `__getitem__`.**

```python
class Deck:
    def __init__(self, cards): self._cards = list(cards)
    def __len__(self): return len(self._cards)
    def __getitem__(self, i): return self._cards[i]     # this ONE method gives you a lot

d = Deck(["A", "K", "Q"])
len(d)              # 3
d[0]                # 'A'
d[::2]              # ['A', 'Q']   — slicing, because list.__getitem__ handles slices
for c in d: ...     # iteration — the legacy protocol falls back to __getitem__(0), (1)...
"K" in d            # True — falls back to iteration
reversed(d)         # works, via __len__ + __getitem__
```

This is the classic *Fluent Python* demonstration: two dunders and your class is a first-class sequence. Note the fallbacks — `in` uses `__contains__` if present, else iterates; `iter()` uses `__iter__` if present, else the old `__getitem__(0..n)` protocol.

**`__call__`** — makes an instance behave like a function. This is how class-based decorators work (Lesson 04) and how stateful "function objects" get built.

**Context manager dunders** (`__enter__`/`__exit__`) get their own lesson (08).

### 2.5 `@property` — the reason Python has no getters

Start with a plain attribute. **Always start with a plain attribute.** When you later need validation or computation, `@property` upgrades it with zero call-site changes:

```python
class Account:
    def __init__(self, balance):
        self.balance = balance          # this ALREADY calls the setter below

    @property
    def balance(self):                  # getter
        return self._balance

    @balance.setter
    def balance(self, value):           # validation, added later, no caller changes
        if value < 0:
            raise ValueError(f"balance cannot be negative: {value}")
        self._balance = value

    @property
    def is_overdrawn(self):             # computed, read-only — no setter defined
        return self._balance < 0

a = Account(100)
a.balance = 50        # calls the setter
a.balance             # 50 — calls the getter
a.balance = -1        # ValueError
a.is_overdrawn = True # AttributeError: property 'is_overdrawn' has no setter
```

The `self.balance = balance` in `__init__` going through the setter is the elegant part — validation happens on construction *and* on every later assignment, with one implementation.

**`functools.cached_property`** computes once and then **replaces itself** in the instance `__dict__`:

```python
from functools import cached_property

class Report:
    @cached_property
    def rows(self):
        print("expensive query!")
        return fetch_from_db()

r = Report()
r.rows      # "expensive query!" then the rows
r.rows      # just the rows — the property is gone, it's an instance attr now
```

It works because `cached_property` is a **non-data descriptor** (no `__set__`), so once the instance dict has the key, step 2 of attribute lookup wins and the descriptor is never consulted again. That's the whole trick — and it's why `cached_property` needs a `__dict__` and therefore **doesn't work with `__slots__`**. Also: no invalidation, and not thread-safe for the compute (two threads can both compute).

### 2.6 Descriptors — the machinery under everything

A descriptor is any object defining `__get__`, `__set__`, or `__delete__`, used as a **class attribute**.

- **Data descriptor**: defines `__set__` and/or `__delete__` → **takes precedence over the instance dict**.
- **Non-data descriptor**: only `__get__` → **the instance dict wins**.

```python
class Positive:
    def __set_name__(self, owner, name):        # 3.6+: told its own attribute name
        self._name = f"_{name}"
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self                          # accessed on the CLASS, not an instance
        return getattr(obj, self._name)
    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self._name[1:]} must be positive, got {value}")
        setattr(obj, self._name, value)

class Product:
    price = Positive()          # reusable validation, declared once
    quantity = Positive()
    def __init__(self, price, quantity):
        self.price = price      # goes through Positive.__set__
        self.quantity = quantity

p = Product(10, 5)
p.price = -1                    # ValueError: price must be positive, got -1
Product.price                   # <Positive object> — because obj is None in __get__
```

Now the revelation: **things you use daily are descriptors.**

| Thing | Kind | What its `__get__` does |
|---|---|---|
| a plain function | non-data | returns a **bound method** — this is how `self` gets passed! |
| `property` | **data** | calls your getter/setter |
| `classmethod` | non-data | returns a method bound to the *class* |
| `staticmethod` | non-data | returns the plain function, unbound |
| `__slots__` entries | **data** | reads/writes the fixed C array slot |
| `functools.cached_property` | non-data | computes, then writes to `obj.__dict__` |
| SQLAlchemy `Column`, Pydantic fields, Django model fields | data | build SQL / validate |

That first row is the big one. **`self` is not magic** — `obj.method` is `type(obj).__dict__['method'].__get__(obj, type(obj))`, which returns a `MethodType` object that closes over `obj` and passes it as the first argument. Explaining this is a top-tier interview answer.

### 2.7 `@staticmethod` vs `@classmethod` vs module function

```python
class Pizza:
    def __init__(self, radius, toppings):
        self.radius, self.toppings = radius, toppings

    def area(self):                                  # instance method — needs self
        return self._circle_area(self.radius)

    @classmethod
    def margherita(cls):                             # ALTERNATIVE CONSTRUCTOR — the killer use
        return cls(12, ["mozzarella", "tomato"])     # 'cls', so subclasses get their own type

    @staticmethod
    def _circle_area(r):                             # no self, no cls — just namespaced
        return math.pi * r ** 2
```

- **`classmethod`** receives the class. Its dominant use is the **alternative constructor** (`dict.fromkeys`, `datetime.now`, `Model.from_json`). Using `cls(...)` rather than `Pizza(...)` means `class SicilianPizza(Pizza)` inherits `margherita()` and gets a `SicilianPizza` back — the whole point.
- **`staticmethod`** receives nothing. It's a function that happens to live in the class namespace. It's honest signalling ("this doesn't touch state") but a module-level function is usually just as good. Use it when the function is meaningless outside the class's context and you want it found by `Pizza._circle_area`.
- **Module-level function** is the Python default. Don't create a class just to hold functions — that's Java-brain. A module *is* a namespace.

### 2.8 Privacy: there isn't any

```python
class Widget:
    def __init__(self):
        self.public = 1
        self._internal = 2       # CONVENTION: "don't touch, not part of the API"
        self.__mangled = 3       # NAME MANGLING: becomes self._Widget__mangled

w = Widget()
w._internal          # 2 — nothing stops you
w.__mangled          # AttributeError
w._Widget__mangled   # 3 — trivially reachable
```

- **`_single`** is a convention. Linters and `from x import *` respect it; the interpreter doesn't. This is the right tool ~99% of the time. "We're all consenting adults here."
- **`__double`** triggers **name mangling** — the compiler rewrites it to `_ClassName__double`. This is **not** for privacy; it's to prevent *accidental name collisions in subclasses*. Use it only in classes designed for inheritance where a subclass might plausibly reuse the name. Overusing it makes debugging and testing miserable.

---

## 3. Official documentation quotes

> "Class definitions, like function definitions (`def` statements) must be executed before they have any effect. … When a class definition is entered, a new namespace is created, and used as the local scope."
> — *Python tutorial*, [9.3.1. Class Definition Syntax](https://docs.python.org/3/tutorial/classes.html#class-definition-syntax)

> "Called by the built-in function `repr()` to compute the 'official' string representation of an object. If at all possible, this should look like a valid Python expression that could be used to recreate an object with the same value. … This is typically used for debugging, so it is important that the representation is information-rich and unambiguous."
> — *The Python Language Reference*, [3.3.1. `object.__repr__`](https://docs.python.org/3/reference/datamodel.html#object.__repr__)

> "If a class that overrides `__eq__()` needs to retain the hash implementation from a parent class, the interpreter must be told this explicitly by setting `__hash__ = <ParentClass>.__hash__`. If a class that does not override `__eq__()` wishes to suppress hash support, it should include `__hash__ = None` in the class definition."
> — *The Python Language Reference*, [3.3.1. `object.__hash__`](https://docs.python.org/3/reference/datamodel.html#object.__hash__)

> "The default behavior for attribute access is to get, set, or delete the attribute from an object's dictionary. … `a.x` has a lookup chain starting with `a.__dict__['x']`, then `type(a).__dict__['x']`, and continuing through the base classes of `type(a)` excluding metaclasses. **If the looked-up value is an object defining one of the descriptor methods, then Python may override the default behavior and invoke the descriptor method instead.**"
> — *Python HOWTOs*, [Descriptor HowTo Guide](https://docs.python.org/3/howto/descriptor.html)

> "Private name mangling: When an identifier that textually occurs in a class definition begins with two or more underscore characters and does not end in two or more underscores, it is considered a private name of that class. … This mangling is done without regard to the syntactic position of the identifier, as long as it occurs within the definition of a class."
> — *The Python Language Reference*, [6.2.1. Identifiers (Names)](https://docs.python.org/3/reference/expressions.html#atom-identifiers)

---

## 4. Real-world examples

### 4.1 A well-behaved value object

```python
import math

class Money:
    """An immutable money amount. Compare, hash, add — behaves like a number."""

    __slots__ = ("_amount_cents", "_currency")

    def __init__(self, amount_cents: int, currency: str = "USD"):
        if not isinstance(amount_cents, int):
            raise TypeError("use integer cents — never floats for money")
        object.__setattr__(self, "_amount_cents", amount_cents)
        object.__setattr__(self, "_currency", currency.upper())

    # --- read-only public API -------------------------------------------
    @property
    def amount_cents(self) -> int:
        return self._amount_cents

    @property
    def currency(self) -> str:
        return self._currency

    @property
    def dollars(self) -> float:
        return self._amount_cents / 100

    # --- alternative constructor ----------------------------------------
    @classmethod
    def from_dollars(cls, dollars: str, currency: str = "USD") -> "Money":
        from decimal import Decimal
        return cls(int(Decimal(dollars) * 100), currency)

    # --- the data model -------------------------------------------------
    def __repr__(self):
        return f"Money({self._amount_cents!r}, {self._currency!r})"

    def __str__(self):
        return f"{self._amount_cents / 100:,.2f} {self._currency}"

    def __eq__(self, other):
        if not isinstance(other, Money):
            return NotImplemented
        return (self._amount_cents, self._currency) == (other._amount_cents, other._currency)

    def __hash__(self):
        return hash((self._amount_cents, self._currency))

    def __lt__(self, other):
        if not isinstance(other, Money) or other._currency != self._currency:
            return NotImplemented
        return self._amount_cents < other._amount_cents

    def __add__(self, other):
        if not isinstance(other, Money):
            return NotImplemented
        if other._currency != self._currency:
            raise ValueError(f"cannot add {self._currency} to {other._currency}")
        return Money(self._amount_cents + other._amount_cents, self._currency)

    def __bool__(self):
        return self._amount_cents != 0
```
```python
>>> a = Money.from_dollars("19.99")
>>> b = Money(1)
>>> a + b
Money(2000, 'USD')
>>> print(a + b)
20.00 USD
>>> sorted([Money(300), Money(100), Money(200)])
[Money(100, 'USD'), Money(200, 'USD'), Money(300, 'USD')]
>>> {Money(100), Money(100)}
{Money(100, 'USD')}
>>> a + Money(100, "EUR")
ValueError: cannot add USD to EUR
```

Everything here is deliberate: integer cents (floats lose money — `0.1 + 0.2 != 0.3`), `NotImplemented` for foreign types, hash over the same fields as `__eq__`, `__slots__` because you might hold millions, and `__lt__` enabling `sorted()` (with `functools.total_ordering` you'd get `<=`, `>`, `>=` free from `__lt__` + `__eq__`).

### 4.2 `@property` added *later*, breaking nothing

```python
# v1 — shipped, in use by 40 call sites
class User:
    def __init__(self, email):
        self.email = email

# v2 — six months later, we need validation and normalisation.
# ZERO call sites change.
class User:
    def __init__(self, email):
        self.email = email                    # goes through the setter

    @property
    def email(self):
        return self._email

    @email.setter
    def email(self, value):
        value = value.strip().lower()
        if "@" not in value:
            raise ValueError(f"invalid email: {value!r}")
        self._email = value

    @property
    def domain(self):                         # new computed attribute, reads like data
        return self._email.split("@", 1)[1]
```
```python
>>> u = User("  Ann@Example.COM ")
>>> u.email
'ann@example.com'
>>> u.domain
'example.com'
>>> u.email = "nope"
ValueError: invalid email: 'nope'
```

**This is the entire argument against getters/setters in Python.** In Java you write `getEmail()` from day one *because you might need validation later*. In Python you never pay that tax — `@property` is the escape hatch, applied only when needed. Say this in an interview when asked "why doesn't Python have private fields?"

### 4.3 A reusable validating descriptor

```python
class Validated:
    """Base for typed, validated attributes."""
    def __set_name__(self, owner, name):
        self.name = name
        self.private = f"_{name}"
    def __get__(self, obj, objtype=None):
        return self if obj is None else getattr(obj, self.private)
    def __set__(self, obj, value):
        self.validate(value)
        setattr(obj, self.private, value)
    def validate(self, value):
        raise NotImplementedError

class Integer(Validated):
    def __init__(self, minvalue=None, maxvalue=None):
        self.minvalue, self.maxvalue = minvalue, maxvalue
    def validate(self, value):
        if not isinstance(value, int):
            raise TypeError(f"{self.name} must be an int, got {type(value).__name__}")
        if self.minvalue is not None and value < self.minvalue:
            raise ValueError(f"{self.name} must be >= {self.minvalue}")
        if self.maxvalue is not None and value > self.maxvalue:
            raise ValueError(f"{self.name} must be <= {self.maxvalue}")

class NonBlank(Validated):
    def validate(self, value):
        if not isinstance(value, str) or not value.strip():
            raise ValueError(f"{self.name} must be a non-empty string")

class Employee:
    name = NonBlank()
    age  = Integer(minvalue=16, maxvalue=100)
    def __init__(self, name, age):
        self.name, self.age = name, age
```
```python
>>> Employee("Ann", 30)
>>> Employee("", 30)
ValueError: name must be a non-empty string
>>> Employee("Ann", 200)
ValueError: age must be <= 100
>>> Employee("Ann", "30")
TypeError: age must be an int, got str
```

Validation declared once, reused across every class, error messages that name the field automatically via `__set_name__`. **This is a hand-rolled Pydantic.** When you next read Pydantic or a SQLAlchemy model, you're reading this pattern with more features. Understanding it is the difference between using an ORM and knowing what it does.

### 4.4 Proving `self` comes from the descriptor protocol

```python
class C:
    def method(self): return f"called on {self}"
    def __repr__(self): return "<C instance>"

c = C()
C.__dict__["method"]                              # <function C.method> — a plain function
C.__dict__["method"].__get__(c, C)                # <bound method C.method of <C instance>>
C.__dict__["method"].__get__(c, C)()              # 'called on <C instance>'
c.method()                                        # 'called on <C instance>' — IDENTICAL

c.method.__self__                                 # <C instance> — the bound object
c.method.__func__                                 # <function C.method>
```
There is no magic in `self`. `c.method` triggers `function.__get__`, which returns a `MethodType` wrapping `(func, c)`; calling it prepends `c`. Once you see this, `@staticmethod` (a descriptor whose `__get__` returns the bare function) and `@classmethod` (whose `__get__` binds the class) stop being special cases.

---

## 5. How This Is Used In Production

- **Startups:** classes model domain concepts (`Order`, `Subscription`, `Money`) and `@property` handles derived values (`order.total`, `subscription.is_active`) so business rules live in one place instead of being recomputed in three templates. A good `__repr__` is disproportionately valuable at this stage — it's what you see in Sentry and in the logs at 2 a.m.
- **Enterprises:** the descriptor protocol *is* the framework layer. SQLAlchemy's declarative models, Django's `Field`s, Pydantic's `FieldInfo`, and attrs/dataclass machinery all use descriptors + `__set_name__` to give you declarative schemas. Enterprises additionally lean on `__slots__` for high-instance-count objects, and on `__eq__`/`__hash__` correctness because dedup and caching layers depend on it — a wrong `__hash__` produces silent data corruption, not a crash.
- **Common architectures:** **value objects** (immutable, `__eq__`/`__hash__` by value — `Money`, `EmailAddress`, `Coordinate`) vs **entities** (identity-based `__eq__` on an ID, mutable state) is the core distinction from Domain-Driven Design, and it's implemented entirely with the dunders in this lesson. The **repository pattern** returns entities; **DTOs** at the boundary are usually Pydantic models or frozen dataclasses. *Architecture Patterns with Python* (Percival & Gregory) is the book-length version of this paragraph.
- **Scaling:** `__slots__` on hot objects halves memory (Lesson 05, §4.5). `cached_property` avoids recomputation but pins memory for the instance's lifetime. `__eq__`/`__hash__` are on the hot path of every set/dict operation — a `__hash__` that hashes a large tuple of strings on every lookup is a real bottleneck; cache it if the object is immutable. Property getters that hit the database (the classic `@property def orders(self): return db.query(...)`) turn `for u in users: print(u.orders)` into an **N+1 query storm** — the most common performance bug in ORM code, and it's caused by making an expensive thing *look* like an attribute.
- **Monitoring & logging:** structured logging depends on `__repr__`. Sentry, Datadog, and every APM serialise objects via `repr` when capturing local variables in a traceback — an object with the default `<Foo object at 0x7f3a>` repr makes the exception report useless. Conversely, a `__repr__` that includes a password, token, or PII **leaks it into your log aggregator**, which is a real compliance incident (see Security).
- **Security:** two concrete issues. (1) **Secrets in `__repr__`** — never include credentials; Pydantic ships `SecretStr` precisely for this, whose `__repr__` prints `**********`. (2) **`@property` that raises** in a `__repr__` path can mask the original exception during error handling. Also note that `_private` is not access control — never rely on it as a security boundary, because it isn't one.
- **CI/CD:** `__eq__` makes tests readable (`assert result == Money(1999)` instead of field-by-field assertions), and a good `__repr__` makes pytest's assertion diffs actually useful. `dataclass`/`attrs` generate both, which is why they dominate test fixtures.
- **Common tools:** `dataclasses` (stdlib — generates `__init__`, `__repr__`, `__eq__`, optionally `__hash__` and ordering), `attrs` (the predecessor, still richer: validators, converters, slots by default), `pydantic` (validation + serialisation, descriptor-driven), `functools.total_ordering` and `cached_property`, `typing.Protocol` (structural typing — Lesson 07).
- **Real examples:** **Instagram**'s Django models and **Netflix**'s internal Python tooling are descriptor-heavy by construction. **Stripe**'s and **Twilio**'s Python SDKs use properties to lazily hydrate nested API resources, which is elegant but is exactly the pattern that surprises users with a network call behind an attribute access — the SDK docs have to warn about it. That trade-off (attribute syntax hiding expensive work) is a genuinely good design-discussion topic in interviews.

---

## 6. Advanced corner

**The MRO and C3 linearisation.** `type(obj).__mro__` is the ordered tuple of classes searched for attributes. Python computes it with **C3 linearisation**, which guarantees: a class precedes its parents, parents appear in the order listed, and the ordering is monotonic (consistent across the hierarchy). If no consistent order exists, the `class` statement itself raises `TypeError: Cannot create a consistent method resolution order`. `super()` does **not** mean "my parent" — it means "the next class in the MRO **of the instance's type**," which is why cooperative multiple inheritance works and why `super()` in a diamond can call a *sibling* class you never inherited from directly. (Full treatment in Lesson 07.)

**`super()` with no arguments is compiler magic.** It relies on `__class__` being injected as an implicit closure cell in any method that mentions `super` or `__class__` — you can see it in `method.__code__.co_freevars`. This is why zero-arg `super()` fails outside a class body and why `super(Foo, self)` still exists for edge cases.

**`__getattr__` vs `__getattribute__`.** `__getattribute__` intercepts **every** attribute access, including dunders looked up on the instance (though *implicit* dunder lookup for operators bypasses it — see below). Overriding it is a footgun: `self.x` inside it recurses infinitely; you must call `object.__getattribute__(self, name)`. `__getattr__` fires only after normal lookup raises `AttributeError`, which makes it safe and cheap — and it's how lazy proxies, mock objects, and dynamic API clients are built. Note that `__getattr__` swallowing everything breaks `hasattr`, `copy`, and `pickle`, which probe for dunders like `__deepcopy__` and `__reduce__`; a `__getattr__` returning a Mock for those causes bizarre bugs. Always raise `AttributeError` for names starting and ending with `__`.

**Implicit dunder lookup skips the instance.** `len(obj)` calls `type(obj).__len__(obj)`, **not** `obj.__len__()`. So setting `obj.__len__ = lambda: 5` on an instance does nothing.
```python
class C: pass
c = C()
c.__len__ = lambda: 5
len(c)             # TypeError: object of type 'C' has no len()
c.__len__()        # 5 — the attribute is there, len() just doesn't look at it
```
This is a real design decision (it makes operators fast and keeps `type(x) is type(y)` implying the same operator behaviour), and it's why you can't monkey-patch dunders per instance. It's also a favourite senior interview question.

**`__slots__` interacts with everything.** Slots are data descriptors on the class. Adding `__slots__` removes `__dict__`, which breaks: `cached_property`, arbitrary attribute assignment, `weakref` (unless you add `"__weakref__"`), pickling in old protocols, and multiple inheritance from two slotted classes with overlapping layouts. A subclass without `__slots__` silently regains a `__dict__` — so `class B(SlottedA): pass` throws away all the savings, a mistake that's near-invisible in review.

**`__eq__` performance.** Every set/dict operation calls `__hash__` then, on a bucket hit, `__eq__`. A `dataclass`-generated `__eq__` builds a tuple of all fields on **every** comparison — fine for a handful of fields, measurably bad for hot loops with wide objects. Compare an ID field first when you have one (`if self.id != other.id: return False`) or use `eq=False` and hand-write it.

**Class attribute lookup is cached.** CPython maintains a per-type method cache invalidated by a global "type modification" counter; monkey-patching a class at runtime invalidates it process-wide. This is one reason runtime monkey-patching in hot paths performs worse than people expect. (3.12's per-type versioning made this much better.)

**What most people get wrong at this level:**
- Writing Java in Python: getters/setters from day one, classes as function bags, `AbstractSingletonFactory`. Python's answer is a plain attribute, a module, and a function.
- Defining `__eq__` and forgetting `__hash__`, then wondering why the object can't go in a set.
- Returning `False` instead of `NotImplemented` from `__eq__`, quietly breaking comparison with types that would have handled it.
- Believing `__double` is private. It's mangled, not hidden.
- Thinking `@property` is free — an expensive property is a landmine, because callers reasonably assume attribute access is O(1) and cheap. That assumption is why N+1 queries happen.
- Using `__new__` when they meant `__init__`.
- Not writing `__repr__`, then losing an hour in a debugger.

---

## 7. Best practices

- **Write `__repr__` for every non-trivial class.** Make it unambiguous, include the identifying fields, use `!r` on them, and never include secrets. If you write only one string dunder, write this one.
- **Start with public attributes.** Add `@property` only when you need validation, computation, or a deprecation shim. Never write `get_x()`/`set_x()` in Python.
- **`__eq__` and `__hash__` travel together.** Hash the same fields you compare; make them immutable; return `NotImplemented` for foreign types. Better yet, use `@dataclass(frozen=True)` and let Python generate both correctly.
- **Prefer `@dataclass` / `attrs` over hand-writing boilerplate.** Hand-write only when you need behaviour the generators can't express (like `Money.__add__`).
- **Keep properties cheap and side-effect-free.** If it does I/O, name it `fetch_orders()` — a method — so the cost is visible at the call site. This single rule prevents most N+1 bugs.
- Use `@classmethod` for **alternative constructors**, always with `cls(...)` not the hard-coded class name.
- Prefer a module-level function to `@staticmethod` unless the class name adds real context.
- Use `_single_underscore` for internals; reserve `__double` for genuine subclass-collision risk in library base classes.
- Add `__slots__` when you'll have >100k instances — and add it to **every** class in the hierarchy or you get no benefit.
- Use `functools.total_ordering` to get all six comparisons from `__eq__` + `__lt__`.
- Don't override `__getattribute__`. Reach for `__getattr__`, and always raise `AttributeError` for dunder names inside it.

## 8. Common mistakes & gotchas

**Mutable class attribute** — the Lesson-04 default-argument trap wearing a different hat:
```python
class Cart:
    items = []                      # ONE list for ALL carts
a, b = Cart(), Cart()
a.items.append("x")
b.items                             # ['x']  — surprise
```
Fix: assign in `__init__`. Note the asymmetry: `a.items.append(...)` mutates the *class* attribute, but `a.items = [...]` creates an *instance* attribute that shadows it — so the bug is intermittent depending on whether callers mutate or rebind. That's what makes it so hard to spot.

**`__eq__` without `__hash__`**
```python
class P:
    def __init__(self, x): self.x = x
    def __eq__(self, o): return self.x == o.x
{P(1)}                              # TypeError: unhashable type: 'P'
```

**Returning `False` from `__eq__` for unknown types** — breaks the reflected-operand fallback.

**Expensive property**
```python
class User:
    @property
    def orders(self):
        return db.query(Order).filter_by(user_id=self.id).all()   # a QUERY, behind a dot

for u in users:        # 1 query for users...
    print(u.orders)    # ...+ N queries. The classic N+1.
```

**`self.x = 1` inside a property setter for `x`** → infinite recursion. Use a different backing name (`self._x`).

**Instance dunders are ignored** — `c.__len__ = ...` doesn't make `len(c)` work (§6).

**`__slots__` defeated by a subclass** — `class B(SlottedA): pass` reintroduces `__dict__`.

**`cached_property` never invalidates** — the data goes stale and there's no API to refresh it. `del obj.attr` is the (undocumented-feeling but correct) way to bust it.

**Class body can't see class scope from a comprehension** (Lesson 04, §8) — `[x + sep for x in names]` in a class body raises `NameError` for `sep`.

**`__init__` returning something** — `TypeError: __init__() should return None`.

## 9. Where AI helps (and where it hurts)

- **Helps:** generating the dunder boilerplate you'd otherwise fat-finger (`__eq__`/`__hash__`/`__repr__`/`__lt__` over a field list); explaining an unfamiliar descriptor in a framework's source; converting a hand-rolled class to a `dataclass`; producing the `__set_name__` plumbing correctly; writing the "prove `self` is a descriptor" demos.
- **Hurts:** AI writes **Java-shaped Python** by default — it will hand you `get_name()`/`set_name()` and a `Factory` class unless you tell it not to, because its training data is dominated by OOP conventions from other languages. It routinely defines `__eq__` without `__hash__`. It suggests `__new__` for things `__init__` does. And it will confidently tell you `__double_underscore` is "private," which is wrong and will mislead you in an interview. It's genuinely good at *explaining* descriptors once you ask, and genuinely bad at *deciding* whether you need one — the answer is usually "no, use a dataclass."

## 10. Learn independently

**Books (specific chapters):**
- *Fluent Python*, 2nd ed. (Ramalho) — **Ch. 1 "The Python Data Model"** (the `FrenchDeck` example — read this first, it's the best 20 pages in Python literature), **Ch. 11 "A Pythonic Object"** (`__repr__`, `__eq__`, `__hash__`, `__slots__`, private attrs — essentially this lesson), **Ch. 12 "Special Methods for Sequences"**, **Ch. 22 "Dynamic Attributes and Properties"** (`__getattr__`, properties, `__set_name__`), **Ch. 23 "Attribute Descriptors"** (the definitive treatment). Chapters 11 and 23 are the core.
- *Effective Python*, 2nd ed. (Slatkin) — **Item 37** (compose classes instead of nesting dicts/tuples), **Item 44** ("Use Plain Attributes Instead of Setter and Getter Methods"), **Item 45** ("Consider `@property` Instead of Refactoring Attributes"), **Item 46** ("Use Descriptors for Reusable `@property` Methods"), **Item 47** (`__getattr__`, `__getattribute__`, `__setattr__` for lazy attributes). Items 44–47 are exactly this lesson.
- *Python Cookbook*, 3rd ed. (Beazley & Jones) — **Ch. 8 "Classes and Objects"** — 25 recipes: 8.1 (`__repr__`), 8.6 (managed attributes), 8.10 (lazily computed properties), 8.13 (a data model with type validation via descriptors), 8.20 (calling methods by name).
- *Architecture Patterns with Python* (Percival & Gregory) — **Ch. 1 "Domain Modeling"** — value objects vs entities, and why `__eq__`/`__hash__` choice *is* a design decision. Free at [cosmicpython.com](https://www.cosmicpython.com/).
- *CPython Internals* (Shaw) — the objects/type chapters for `tp_getattro` and how the C level implements all of this.

**Primary docs:**
- [Data model reference](https://docs.python.org/3/reference/datamodel.html) — the canonical list of every dunder. Bookmark it; you'll return for years.
- [**Descriptor HowTo Guide**](https://docs.python.org/3/howto/descriptor.html) — Raymond Hettinger's guide, including pure-Python reimplementations of `property`, `classmethod`, `staticmethod`, and functions-as-descriptors. **Required reading.** Work through it with the code in a REPL.
- [`functools.cached_property`](https://docs.python.org/3/library/functools.html#functools.cached_property) and [`total_ordering`](https://docs.python.org/3/library/functools.html#functools.total_ordering)
- [Classes tutorial](https://docs.python.org/3/tutorial/classes.html) — §9.6 on private variables states the mangling rule precisely.

**High-quality links:**
- Raymond Hettinger, *"Python's Class Development Toolkit"* (PyCon 2013) — properties, class methods, slots, in one talk. Timeless.
- Raymond Hettinger, *"Super Considered Super!"* — for Lesson 07, but the MRO explanation applies here.
- [cosmicpython.com](https://www.cosmicpython.com/) — the full DDD-in-Python book, free.

## 11. Interview preparation

**Q1. `__str__` vs `__repr__` — which do you implement?**
> `__repr__`, always; `__str__` only if the user-facing form differs. `__repr__` is the unambiguous developer representation — ideally something you could paste back into a REPL — and it's what you see in the REPL, in logs, in tracebacks, and inside containers (`[obj]` uses `repr` on the elements, always). `__str__` falls back to `__repr__` if undefined, so implementing `__repr__` gives you both. The practical argument: a missing `__repr__` means your Sentry reports say `<User object at 0x7f3a>`, which costs you real time at 2 a.m.

**Q2. Why does defining `__eq__` make my class unhashable?**
> Python sets `__hash__ = None` automatically. It has to: the inherited default hash is based on `id()`, so two objects you've just declared equal would hash differently — violating the contract that `a == b` implies `hash(a) == hash(b)`, which would break every set and dict silently rather than loudly. Python forces you to make the decision. Define `__hash__` over the same fields you compare (and make them immutable), or explicitly set `__hash__ = None` if the object should stay unhashable, or use `@dataclass(frozen=True)` which does it correctly for you.

**Q3. What is a descriptor?**
> Any object implementing `__get__`, `__set__`, or `__delete__`, used as a **class attribute**. It lets the class intercept attribute access on its instances. **Data descriptors** (with `__set__`/`__delete__`) take priority over the instance `__dict__`; **non-data descriptors** (only `__get__`) lose to it. That precedence rule is the whole mechanism: it's why `@property` can't be shadowed by `self.x = ...`, and why `cached_property` — a non-data descriptor — can write into the instance dict and thereby remove itself from the lookup path on subsequent accesses. Descriptors are also the answer to "how does `self` get passed": functions are non-data descriptors whose `__get__` returns a bound method.

**Q4. `@staticmethod` vs `@classmethod` vs a module function?**
> `classmethod` gets the class as the first argument — its main use is the **alternative constructor** (`Model.from_json`, `datetime.now`), and using `cls(...)` rather than the literal class name means subclasses inherit it correctly and get their own type back. `staticmethod` gets nothing; it's a function namespaced inside the class. A module-level function is the Python default — a module already *is* a namespace, so don't create a class just to hold functions. I use `staticmethod` only when the function is genuinely meaningless outside the class's context and I want it discoverable as `Cls.helper`.

**Q5. Are there private attributes in Python?**
> No. `_single` is a convention — linters, `import *`, and IDEs respect it, the interpreter doesn't. `__double` triggers **name mangling**: the compiler rewrites it to `_ClassName__double`, which is trivially accessible if you know the rule. Mangling exists to prevent *accidental* name collisions in subclasses, not to enforce access control. The philosophy is "we're all consenting adults" — combined with `@property`, it means you can start with a public attribute and add validation later without changing callers, which is exactly the flexibility that makes Java-style getters unnecessary.

**Q6. `__new__` vs `__init__`?**
> `__new__` is the static allocator — it creates and returns the instance. `__init__` is the initialiser — it configures an already-created instance and must return `None`. `Foo(x)` invokes `type.__call__`, which calls `__new__`, then calls `__init__` **only if `__new__` returned an instance of `Foo`**. You need `__new__` in three cases: subclassing an immutable builtin (`int`, `str`, `tuple` — their value is set at allocation, so `__init__` is too late), instance caching/singletons, and metaclass work. Otherwise use `__init__`. The gotcha: return the wrong type from `__new__` and `__init__` silently never runs.

**Q7. What's the difference between `__getattr__` and `__getattribute__`?**
> `__getattribute__` intercepts **every** attribute access; `__getattr__` is only called **after** normal lookup has failed with `AttributeError`. So `__getattr__` is a safe fallback hook — used for lazy proxies, dynamic API clients, mocks — while `__getattribute__` is a dangerous total override that recurses infinitely if you write `self.x` inside it (you must call `object.__getattribute__`). One caveat for `__getattr__`: always raise `AttributeError` for names like `__deepcopy__`/`__reduce__`, or you'll silently break `copy`, `pickle`, and `hasattr` — a bug that presents as something entirely unrelated.

**Q8. Explain why `len(obj)` doesn't see `obj.__len__` set on the instance.**
> Implicit special-method lookup **skips the instance and goes straight to the type**: `len(obj)` is `type(obj).__len__(obj)`. Setting `obj.__len__ = lambda: 5` puts an entry in the instance dict that `len()` never consults, so `len(obj)` raises `TypeError` while `obj.__len__()` returns 5. This is deliberate — it lets CPython cache operator slots on the type for speed, and it guarantees that all instances of a type behave identically under operators. The practical implication: you cannot monkey-patch dunders per instance; patch the class, or use a subclass.

**Q9. (Trade-off) When is `@property` the wrong tool?**
> When it isn't cheap. Attribute syntax is a *promise* to the caller that access is fast and side-effect-free; the moment `user.orders` runs a query, callers will write `for u in users: print(u.orders)` and produce an N+1 — not because they're careless, but because the API told them it was free. If it does I/O, name it `fetch_orders()` so the cost is visible at the call site. Properties are also wrong when you need parameters (that's a method), when the computation can fail in interesting ways (an exception from a dot access is surprising, and it's especially nasty inside `__repr__` during error handling), and when a plain attribute would do — don't wrap a field in a property that just returns it "for future-proofing," because `@property` exists precisely so you don't have to.

**Q10. (System design) Design a `Money` type. What decisions matter?**
> **Integer minor units, never float** — binary floats can't represent `0.10`, so `0.1 + 0.2 != 0.3` and errors compound across a ledger; store cents as `int`, or use `Decimal` if you need sub-cent precision. **Currency is part of the value** — `__eq__` and `__hash__` must include it, and `__add__` must refuse to mix currencies (raise, don't silently convert; conversion needs a rate and a timestamp, so it belongs in a service, not an operator). **Immutable** — it's a value object, so `frozen=True` or `__slots__` + read-only properties; that makes it safely hashable and shareable without defensive copies. **`NotImplemented` for foreign types** so comparison with other numeric types can still work via reflection. **`__lt__` + `total_ordering`** for sorting, but only within a currency — cross-currency ordering is meaningless and should raise. Finally, `__repr__` shows the exact stored value (`Money(1999, 'USD')`) while `__str__` formats for humans (`19.99 USD`). The meta-point: this is a *value object* in DDD terms — equality by value, no identity, no lifecycle — as opposed to an *entity* like `Account`, where equality is by ID and the balance mutates.

**Q11. (Senior) How would you implement a field that validates on assignment across 30 model classes?**
> A **descriptor** — one `Validated` base with `__set_name__`/`__get__`/`__set__`, subclassed per rule (`Integer(min=0)`, `NonBlank()`), declared as class attributes on each model. `__set_name__` gives each instance its own field name automatically, so error messages are specific for free. This is exactly what Pydantic, SQLAlchemy, and Django Fields do — 30 `@property` pairs would be 60 near-identical methods, which is the smell that says "descriptor." That said, in a real 2026 codebase I'd reach for **Pydantic** first and only hand-roll descriptors if I needed something it can't express, because Pydantic also gives me serialisation, JSON Schema, and a validated settings story I'd otherwise rebuild badly.

**Q12. (Senior) Walk me through what `obj.method()` does, precisely.**
> `obj.method` invokes `type(obj).__getattribute__(obj, "method")`. That walks `type(obj).__mro__` looking for `"method"`; it finds a plain function in the class `__dict__`. A function is a **non-data descriptor** (it has `__get__` but no `__set__`), so — since `"method"` isn't in `obj.__dict__` — Python calls `function.__get__(obj, type(obj))`, which returns a `MethodType` object holding `__func__` (the function) and `__self__` (the instance). Calling *that* prepends `__self__` to the argument list and invokes the underlying function. So `self` isn't a language keyword or compiler magic — it's the descriptor protocol doing partial application. The corollary: `Cls.method(obj)` and `obj.method()` are the same call, which is why `super().method()` and unbound-style invocation work.

## 12. Homework

> Work in `python/solutions/lesson-06/`. Type-annotate everything; run `mypy --strict` on your final code.

**Easy**

1. **`__repr__` discipline.** Write a `Book(title, author, isbn, pages)` with a `__repr__` that is `eval`-able and a `__str__` that's human-readable. Prove: `eval(repr(b)) == b` (you'll need `__eq__`), that `print([b])` uses `repr`, and that `f"{b}"` vs `f"{b!r}"` differ.
   *Acceptance:* a script demonstrating all four behaviours with output.

2. **The class-attribute trap.** Write `BadCart` with `items = []` at class level and `GoodCart` assigning in `__init__`. Show the shared-state bug. Then show the *asymmetry*: `a.items.append("x")` leaks to `b`, but `a.items = ["x"]` does not — and explain why in a comment referencing the attribute-lookup algorithm.
   *Acceptance:* printed evidence of both behaviours plus the written explanation.

3. **Property basics.** Take `class Temperature` storing `_celsius`. Add: a validated `celsius` property (reject below −273.15), a computed read-write `fahrenheit` property (setting it updates `_celsius`), and a read-only `kelvin`. Show that `Temperature(25).fahrenheit == 77.0` and that `t.kelvin = 0` raises `AttributeError`.
   *Acceptance:* pytest covering the conversion both ways, the validation, and the read-only failure.

**Medium**

4. **A complete value object.** Implement `Version(major, minor, patch, prerelease=None)` supporting: `__repr__`/`__str__` (`"1.2.3-rc1"`), `__eq__`/`__hash__`, full ordering via `__lt__` + `@total_ordering` (with correct semver prerelease rules — `1.0.0-rc1 < 1.0.0`), a `Version.parse("1.2.3-rc1")` classmethod, and `__bool__` (falsy for `0.0.0`).
   *Acceptance:* pytest with at least 15 cases including sorting a list of 8 versions into correct semver order, set-deduplication, and `parse` round-tripping through `str`.

5. **Sequence protocol.** Implement `Playlist` wrapping a list of tracks with `__len__`, `__getitem__` (supporting **slices**, returning a `Playlist` not a list), `__contains__` (by track title, case-insensitive), `__iter__`, `__reversed__`, and `__add__` (concatenation). Then delete `__iter__` and `__contains__` and show which still work via the `__getitem__` fallback.
   *Acceptance:* tests for indexing, negative indexing, slicing returning the right type, membership, iteration, and the fallback demonstration written up in `NOTES.md`.

6. **`cached_property` and its costs.** Write a `Report` class with a `cached_property` doing an "expensive" computation (sleep + counter). Prove it computes once. Then: show `del r.rows` busts the cache; show that adding `__slots__` breaks it entirely (with the exact error); show two threads can both compute it (use a barrier). Implement a thread-safe alternative with a lock.
   *Acceptance:* four demonstrations with output, plus a paragraph on when you'd accept the double-compute vs. pay for the lock.

7. **`staticmethod`/`classmethod` inheritance.** Build `Shape` with a `@classmethod unit()` alternative constructor using `cls(...)`, and subclasses `Square`/`Circle`. Show `Square.unit()` returns a `Square`. Then break it by hard-coding `Shape(...)` and show what happens. Add a `@staticmethod` and show it's inherited unchanged.
   *Acceptance:* tests asserting `type(Square.unit()) is Square`, plus the broken version's output documented.

**Hard**

8. **Descriptor library.** Build `Validated` + subclasses `Integer(min, max)`, `Float(min, max)`, `String(minlen, maxlen, pattern)`, `OneOf(*options)`, and `Typed(cls)`. Use `__set_name__` for field names. Then build a `Config` class using all five, and a `Config.from_dict()` classmethod that reports **all** validation errors at once (not just the first).
   *Acceptance:* pytest covering each validator's happy and sad paths, correct field names in every error message, `Config.price` (class access) returning the descriptor, and `from_dict` raising a single exception listing 3 simultaneous failures.

9. **Reimplement `property` in pure Python.** Write a `MyProperty` class with `__get__`/`__set__`/`__delete__`/`__set_name__` plus `.setter`/`.deleter` methods so that `@MyProperty` and `@x.setter` work exactly like the builtin. Then do the same for `MyStaticMethod` and `MyClassMethod`.
   *Acceptance:* a test suite that runs **identically** against your versions and the builtins (parametrise over both). Reference: the Descriptor HowTo Guide — read it *after* attempting this.

10. **Break the hash contract, then fix it.** Build an `Entity` (mutable, `__eq__` by `id` field) and a `ValueObj` (mutable, `__eq__` by all fields, `__hash__` over all fields). For `ValueObj`: insert into a set, mutate a field, show `x in s` is `False` while `x in list(s)` is `True` — the object is both in and not in the set. Then show two `==` objects with different hashes both surviving in one set. Fix with `frozen=True`.
    *Acceptance:* printed evidence of both violations, a written explanation tying each to the specific hash-table mechanic it breaks (bucket location; dedup on insert), and the corrected version.

**Stretch (senior)**

11. **Descriptor vs property vs dataclass — measured.** Implement the same 8-field validated record three ways: 8 `@property` pairs, 8 descriptors, and a Pydantic model. Benchmark construction of 1M instances and attribute reads of 10M. Measure memory with `tracemalloc`. Write up which you'd ship and why, including the lines-of-code count.
    *Acceptance:* a results table (construct µs, read ns, bytes/instance, LOC) and a recommendation paragraph that does *not* simply pick the fastest.

12. **A lazy API-client proxy.** Build `LazyResource` that takes an ID and a fetch function and uses `__getattr__` to hydrate from the "API" (a stub with a call counter) on first unknown-attribute access, caching thereafter. Requirements: exactly one fetch for N attribute reads; `hasattr` works; `repr` works **without** triggering a fetch (show the ID only); `copy.deepcopy` and `pickle.dumps` must not explode.
    *Acceptance:* tests for all five requirements. The `pickle`/`deepcopy` ones will fail first — diagnose why (hint: `__reduce_ex__` probing) and fix by raising `AttributeError` for dunders. Document the diagnosis; it's the real lesson.

## 13. Key takeaways

- `class` **executes** and builds an object via `type(name, bases, ns)`. Class attributes are **shared** — the mutable-class-attribute trap is the mutable-default trap in a costume.
- Attribute lookup: **data descriptor → instance `__dict__` → non-data descriptor/class attr → `__getattr__` → `AttributeError`**. That precedence explains `@property`, `cached_property`, and `__slots__` in one rule.
- **Write `__repr__`.** It's what logs, tracebacks, Sentry, and containers show you.
- `__eq__` and `__hash__` are a **pair**; defining `__eq__` alone sets `__hash__ = None`. Return `NotImplemented`, not `False`, for foreign types.
- **`@property` is why Python has no getters.** Start with a plain attribute; upgrade later with zero call-site churn. But an expensive property is a lie that causes N+1s.
- **Descriptors are the machinery under everything** — `property`, `classmethod`, `staticmethod`, `__slots__`, `cached_property`, and **plain functions** (which is where `self` comes from) are all descriptors. So are SQLAlchemy columns and Pydantic fields.
- `@classmethod` = alternative constructor (use `cls`). `@staticmethod` ≈ a module function that wanted a namespace.
- There is no privacy: `_x` is a convention, `__x` is **name mangling** against subclass collisions — not access control.
- Implicit dunder lookup goes to the **type**, skipping the instance. You cannot patch operators per-instance.
