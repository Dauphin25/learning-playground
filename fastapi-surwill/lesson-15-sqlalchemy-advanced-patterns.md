# Lesson 15 — DEEP DIVE: Advanced SQLAlchemy Patterns from Surwill

**Track:** FastAPI (Surwill) · **Lesson:** 15 of 17 · **Estimated time:** 2.5 hours
**Prerequisites:** Lesson 06 (Async SQLAlchemy & the Data Layer), Lesson 07 (Alembic Migrations), basic SQL joins
**Learning objectives:**
- Model relationships correctly (`relationship()`, `back_populates`, cascade rules) and understand exactly when SQLAlchemy issues a query for them
- Choose between `lazy="raise"`, `selectinload`, `joinedload`, and `lazy="select"` with real trade-off reasoning, not folklore
- Represent money, JSON-shaped data, and soft-delete correctly at the column level
- Read and write a "reuse existing data instead of re-fetching" query — a real deduplication pattern
- Understand the Session's identity map, unit-of-work, and `expire_on_commit` well enough to explain a "DetachedInstanceError" in an interview
- Know where the ORM should get out of the way and raw SQL/Core should take over

---

## Why this matters

Lesson 06 covered the skeleton: the async engine, the session factory, `lazy="raise"` as a philosophy, and the atomic-reservation transaction as the flagship example. That's the right first lesson — but it stops at "how Surwill avoids N+1." It doesn't yet show you *how relationships are declared*, *how cascades protect referential integrity without extra application code*, *how the ORM's identity map changes what "the same row" even means inside a request*, or *how to make a deliberate choice not to add credits-costing scrape traffic when a row already has what you need*.

These are the patterns that separate someone who has "used SQLAlchemy" from someone who can be handed an unfamiliar model file and correctly predict what query it will run, or debug why a `DetachedInstanceError` just appeared in production. That's exactly the level a mid-to-senior interview probes for — not "what does `Session` do" but "walk me through what happens, step by step, when this relationship is accessed after commit."

## Theory

### The mental model: Session as a Unit of Work + Identity Map

SQLAlchemy's `Session` (here, `AsyncSession`) is not a connection wrapper. It's two things layered together:

1. **A Unit of Work.** You mutate Python objects — set an attribute, add a new instance, delete one — and the Session tracks *what changed* without touching the database. Only `flush()` (implicit before every query, and explicit before `commit()`) turns those changes into SQL, and it batches them intelligently (all inserts of one type together, etc.).
2. **An Identity Map.** Within one Session, loading the same primary key twice returns the *same Python object*, not two copies. `select(Item).where(Item.id == x)` run twice in the same session, with no changes in between, gives you `is`-identical objects. This is why mutating an object you got from a query, then querying for it again, "sees" your unsaved change — you're handed the same object back out of the map, not a fresh row from the database.

This matters practically: the identity map is *per Session*, not global and not per-process. Two concurrent requests (two Sessions) can hold two different Python objects for the same database row, each unaware of the other's in-memory changes until one commits and the other's Session either re-reads or has that row expired.

### Declaring relationships: what `relationship()` actually configures

A `relationship()` doesn't create a column. It configures **how the ORM should build a query to fetch related rows**, using an existing foreign key (or an explicit `primaryjoin` if the FK is ambiguous or absent). Everything about *when* that query runs is a separate axis — the `lazy=` parameter.

```python
# app/items/models.py
board: Mapped["Board"] = relationship(back_populates="items", lazy="raise")
reservation: Mapped["Reservation | None"] = relationship(back_populates="item", lazy="raise")
```

```python
# app/boards/models.py
items: Mapped[list["Item"]] = relationship(back_populates="board", lazy="raise")
```

`back_populates` is the two-way binding: setting `item.board = some_board` in Python also appends `item` to `some_board.items` in memory, keeping both sides consistent *without a query* — pure Python bookkeeping. It requires you to name the relationship on both classes and point them at each other explicitly (the older `backref="items"` shorthand auto-generates the reverse side, but `back_populates` is the modern, explicit, type-checker-friendly choice, which is why Surwill uses it everywhere).

### The `lazy=` spectrum, precisely

| `lazy=` | When the query fires | Failure mode if misused |
|---|---|---|
| `"select"` (default) | The **first time** you access the attribute, a **separate** query runs, synchronously from the caller's perspective | N+1: loop over 50 items, access `.board` on each → 50 extra queries |
| `"joined"` | A `LEFT OUTER JOIN` is added to the **original** query, so the related row(s) arrive in the same round trip | Duplicated rows over the network for one-to-many (mitigated by `contains_eager`), wasteful for a relationship you don't always need |
| `"selectin"` | A **second** query, but *one* — `WHERE parent_id IN (...)` for the whole batch of parents already loaded | Still two round trips, but O(1) queries regardless of batch size — usually the best default for one-to-many |
| `"raise"` | Never fires — accessing the attribute without eager-loading it **raises `InvalidRequestError`** | Nothing silently slow; you get a loud, immediate signal at dev time |
| `"noload"` | Always `None`/empty, never queries, no error | Silent data loss if you forget you set this |

Surwill deliberately uses `lazy="raise"` on **every** relationship. This is the single most important ORM decision in the codebase, and it's worth understanding why it beats the "just default to `selectin` everywhere" instinct:

- `selectin` for *every* relationship on *every* query means you're always paying for data you might not need. A `GET /boards/{id}` that only needs the board's own columns still fires a second query for `items` if it's `selectin`.
- `lazy="select"` (the SQLAlchemy default) means N+1 is *possible by accident* — a teammate adds `item.board.owner.username` inside a loop eight months from now, it works fine in a 3-row dev database, and it silently becomes 200 extra queries in production.
- `lazy="raise"` converts "did I forget to eager-load this?" from a production incident into a **local test failure**, the moment you write the code. It forces every relationship access to be a *conscious* decision made at the query site via `.options(selectinload(Item.board))`, where the person writing the query can see exactly what they're paying for.

This is a trade Surwill makes deliberately: a little more verbosity at the query site (you must always specify `.options(...)` if you touch a relationship) in exchange for making expensive-by-accident impossible.

### Cascade rules live in two places, and they mean different things

```python
# app/items/models.py
board_id: Mapped[uuid.UUID] = mapped_column(
    UUID(as_uuid=True), ForeignKey("boards.id", ondelete="CASCADE"), nullable=False, index=True
)
partner_id: Mapped[uuid.UUID | None] = mapped_column(
    UUID(as_uuid=True), ForeignKey("marquee_partners.id", ondelete="SET NULL"), nullable=True, index=True
)
```

`ondelete="CASCADE"` and `ondelete="SET NULL"` are **database-level** cascades, enforced by PostgreSQL itself at the foreign key constraint — they fire even if a row is deleted via raw SQL, a different application, or a psql session, and they fire in a single atomic statement, not N application-level deletes. Surwill's choice here is a small piece of domain modeling: deleting a `Board` should take its `Item`s with it (a board's items have no meaning without the board — `CASCADE`), but deleting a `marquee_partner` (an affiliate/curation link) should leave the `Item` intact, just unlinked (`SET NULL`) — the item the user saved is still theirs.

The ORM has its *own*, separate cascade system (`relationship(cascade="all, delete-orphan")`) that governs what happens to **Python objects already loaded in the session** when you delete the parent object in Python. The two systems overlap in effect but not in mechanism — the DB-level `ondelete` protects you even when the ORM is bypassed; the ORM-level `cascade` only fires through `session.delete()`. Production systems that care about integrity lean on the DB-level constraint as the source of truth and treat ORM cascades as a convenience for the common path.

### Money is `Numeric`, never `Float`

```python
price_estimate: Mapped[Decimal | None] = mapped_column(Numeric(10, 2), nullable=True)
```

`Numeric(10, 2)` maps to PostgreSQL's `NUMERIC(10, 2)` — exact, base-10, fixed-point, 10 total digits with 2 after the decimal point. SQLAlchemy hands you back a Python `Decimal`, not a `float`. This isn't pedantry: `float` is IEEE-754 binary floating point, and `0.1 + 0.2 != 0.3` in binary float — utterly unacceptable for currency, where that error compounds across millions of transactions and can fail an audit. Anywhere money crosses a boundary in Surwill, it's `Decimal` in Python and `Numeric` in Postgres, cast to `str` only at the JSON-response edge (see `WishItemResponse.from_item`, Lesson 04) because `Decimal` doesn't serialize to JSON numbers cleanly by default and float-casting would reintroduce the exact error being avoided.

### JSON-shaped data in a text column — and when to use a real JSON column instead

```python
# app/items/models.py
image_urls: Mapped[str | None] = mapped_column(Text, nullable=True)  # JSON list of gallery image URLs
sizes: Mapped[str | None] = mapped_column(Text, nullable=True)  # JSON list
```

These are small, rarely-queried lists, serialized to a JSON string and parsed back out in the schema layer (`json.loads(existing.image_urls)` — you saw this in the backfill function below, and again in `WishItemResponse.from_item` in Lesson 04). Contrast with `Notification.payload`:

```python
# app/notifications/models.py
payload: Mapped[dict] = mapped_column(JSON, nullable=False, default=dict)
```

`JSON` (PostgreSQL's `json`/`jsonb` via `sqlalchemy.dialects.postgresql.JSON`) is a *real* column type — SQLAlchemy (de)serializes it for you, and more importantly Postgres can **index and query inside it** (`payload[key].astext == value`, which you'll find in `_recent_exists` in `app/notifications/services.py`). The rule of thumb: if you will ever filter or query *inside* the blob, use `JSON`/`JSONB`. If it's opaque data you only ever read whole and never filter by, a text column with manual `json.dumps`/`json.loads` is simpler and avoids committing to a schema-less column you'll regret indexing later. Surwill's `sizes`/`image_urls` are read-whole-or-not-at-all; `Notification.payload` is filtered by key (`payload["item_id"]`) in a dedupe query, so it earns the real `JSON` type.

## Official documentation quotes

> "The unit of work pattern is one of several architectural patterns that are considered part of the domain model pattern family... SQLAlchemy's Session implements this pattern in a comprehensive way." — SQLAlchemy ORM documentation, *Session Basics*, https://docs.sqlalchemy.org/en/20/orm/session_basics.html

> "The `relationship.lazy` parameter... `'select'` — items should be loaded lazily when the property is first accessed, using a separate SELECT statement... `'raise'` — items should never be loaded; instead an exception is raised when the given collection is accessed." — SQLAlchemy ORM documentation, `relationship()` API, https://docs.sqlalchemy.org/en/20/orm/relationship_api.html

> "Identity Map — ... within the scope of a Session, all objects that are loaded within this scope are tracked by primary key. If a query returns a row that matches an object already loaded, the same Python object is returned." — SQLAlchemy ORM documentation, *Session Basics*, https://docs.sqlalchemy.org/en/20/orm/session_basics.html

> "The `NUMERIC` type... generally has performance disadvantages [vs float]... it is generally used for exact fixed-point calculations, such as monetary values." — PostgreSQL documentation, §8.1 Numeric Types, https://www.postgresql.org/docs/current/datatype-numeric.html

## The code, explained

### Reusing data before paying for a scrape — a real deduplication query

```python
# app/items/services.py
async def _backfill_from_existing_item(url: str, data: dict, db: AsyncSession) -> bool:
    """Before paying for a managed scrape, reuse data we already hold.

    If *any* item in the database was previously saved from this exact product
    URL with real data (a title and a price — the placeholder 'Saved item' rows
    from failed scrapes don't count), copy its fields into `data` and report
    True. Product data barely changes day to day, and the hourly refresh keeps
    prices current afterwards — so a second user pasting the same link should
    never cost a credit.
    """
    result = await db.execute(
        select(Item)
        .where(
            Item.product_url == url,
            Item.title != "",
            Item.title != "Saved item",
            Item.price_estimate.isnot(None),
            Item.deleted_at.is_(None),
        )
        .order_by(Item.updated_at.desc())
        .limit(1)
    )
    existing = result.scalar_one_or_none()
    if existing is None:
        return False

    reused: dict = {
        "title": existing.title,
        "description": existing.description,
        "price": str(existing.price_estimate),
        "original_price": str(existing.original_price) if existing.original_price else None,
        "currency": existing.currency,
        "image_url": existing.image_url,
        "brand": existing.brand,
        "source": existing.source,
        "in_stock": existing.in_stock,
    }
    try:
        reused["images"] = json.loads(existing.image_urls) if existing.image_urls else []
    except Exception:
        reused["images"] = []
    ...
```

Walk through what makes this query *correct*, not just functional:

1. **`Item.title != "" / != "Saved item"` and `Item.price_estimate.isnot(None)`** are a *quality filter*, not just an existence check. A row can exist for this URL and still be garbage — a failed scrape that fell all the way through the escalation ladder (Lesson 10) and got saved as a placeholder. Reusing a placeholder would propagate a bad result to a second user instead of giving them a real, paid scrape. This line encodes a business rule ("only reuse *real* data") directly in the WHERE clause, which is the right place for it — it's cheaper to filter in SQL than to fetch a candidate and reject it in Python.
2. **`Item.deleted_at.is_(None)`** — this is Surwill's soft-delete convention (`SoftDeleteMixin`, Lesson 06). Every query against a soft-deletable table must remember this filter; there's no automatic global scope for it here (some ORMs, like Django's default manager, can bake a `WHERE deleted_at IS NULL` into every query automatically — more on that trade-off in Lesson 17).
3. **`.order_by(Item.updated_at.desc()).limit(1)`** — if multiple users saved the same URL, take the *freshest* one. This is a deliberate tie-break, not an arbitrary one: newer data is more likely to reflect the current price/stock than an older duplicate.
4. **`scalar_one_or_none()`** returns the ORM object directly (not a `Row` tuple) or `None` — the right call when your `select()` targets exactly one entity and you expect 0 or 1 rows. Compare to `scalars().all()` (many rows, same shape) and `.first()` (silently takes the first of *however many* rows exist, no guarantee of "at most one" — `scalar_one_or_none` will *raise* if there's more than one, which is a feature: it turns a broken assumption into a loud error instead of a silently-wrong pick).
5. The function reads columns off a **detached-adjacent but still-session-bound** `existing` object and copies primitive values into a plain `dict` (`reused`) before returning. This is important: `data` (the caller's working dict) needs to survive past this function call and this Session's lifetime — copying to plain Python types now avoids ever touching `existing` again after its Session might have expired those attributes on commit (see "Session lifecycle" below).

### Relationships that are declared but *never* casually traversed

Look again at every model file in Surwill and notice what's absent: there is no `for item in board.items: ...` anywhere outside of an explicit `.options(selectinload(Board.items))` query. Every access pattern goes through the *services* layer, which decides up front what it needs and loads it in one shape. This is the practical consequence of `lazy="raise"` — it doesn't just prevent N+1, it forces every relationship traversal to be *visible in a diff*, at the query, rather than buried three calls deep in a template or a serializer.

## How this is used in production

- **Startups** commonly start with `lazy="select"` everywhere (the SQLAlchemy default) because it "just works" in early testing with tiny datasets, and only discover N+1 problems after a customer with a large account complains about a slow page — by which point the pattern is baked into dozens of call sites. Retrofitting `lazy="raise"` onto an existing large codebase is a multi-week project; choosing it from day one (as Surwill does) costs almost nothing.
- **Enterprises** running SQLAlchemy at scale (e.g., large fintech and logistics platforms) typically standardize on `selectin` for one-to-many and `joined` for many-to-one/one-to-one lookups that are almost always needed together, and enforce the choice via code review checklists or custom lint rules rather than `lazy="raise"` — because their codebases are old enough that flipping every relationship to `raise` at once would break hundreds of call sites simultaneously. This is exactly why the *decision to start with `raise`* matters: it's cheap on day one, expensive to retrofit.
- **Money columns**: every payments-adjacent company (Stripe, Shopify, banking backends) uses fixed-point/decimal types for currency, never float, and many go a step further and store the smallest currency unit (cents) as an `Integer` to sidestep decimal-precision questions entirely — `Numeric(10,2)` is the common middle ground when you need to display fractional units directly.
- **Monitoring**: teams that lean on `selectin`/`joined` eager loading typically pair it with query-count assertions in tests (`assert_num_queries(3)` style helpers, common in Django but achievable in SQLAlchemy via SQLAlchemy's event system hooking `before_cursor_execute`) so a regression that reintroduces N+1 fails CI instead of surfacing in a slow-query dashboard weeks later.
- **CI/CD**: Alembic migrations (Lesson 07) that add a new relationship or foreign key are reviewed specifically for the `ondelete` behavior — an accidental default (`ondelete=None`, which means "restrict" at the ORM level but "no action" — effectively blocking the parent delete — at the Postgres level unless the app deletes children first) is one of the most common production incidents in this class of bug: someone can't delete a row and doesn't know why until they read the constraint.

## Advanced corner

**The identity map and `expire_on_commit` — where `DetachedInstanceError` actually comes from.** By default, `Session.commit()` **expires** every object it touched — the next attribute access on any of them triggers a fresh `SELECT` to reload current values, because the Session can no longer assume its in-memory copy matches the database (a concurrent transaction may have changed it). This is *usually* invisible because you're still inside the same request, the Session is still open, and the re-fetch just happens transparently. The classic failure is: you commit, close the Session (end of request), and *then* — in a background task, a `to_dict()` called after the request context is gone, or a lazily-evaluated log statement — you touch an attribute on an object from that closed Session. There's no open connection to refresh from, and you get `DetachedInstanceError`. The fix is never "catch the exception" — it's "extract every primitive value you need into a plain dict/dataclass *before* the Session closes," which is exactly the discipline `_backfill_from_existing_item` follows by building `reused: dict` immediately.

**Why `lazy="raise"` doesn't fully protect you from N+1.** It protects you from *relationship* traversal causing extra queries. It does nothing about a loop that calls `db.execute(select(...))` directly for each iteration — that's still N+1, just written by hand instead of via the ORM's lazy loader. `_persist_item_refresh` in Lesson 09 avoids this by batching: one query to resolve all owners (`Item.id.in_({...})`) instead of one lookup per item. The general principle: *any* per-row query inside a loop is N+1, whether the ORM triggers it implicitly or you write it explicitly — `lazy="raise"` only forces the implicit case to become visible.

**`AsyncSession` and greenlet.** SQLAlchemy's async support is not a native rewrite of the driver stack — `asyncpg` (the actual Postgres driver) is async-native, but a large amount of SQLAlchemy's internal machinery (the ORM's unit-of-work bookkeeping, relationship loading logic) is synchronous code that predates async support by a decade. SQLAlchemy bridges this with `greenlet`, running the sync internals inside a greenlet that can suspend and resume around the async I/O boundary. This is why `AsyncSession` methods are `await`-able even though most of the code they call underneath is not `async def` — it's a genuinely clever piece of engineering, and also why certain synchronous-only APIs (some legacy `Session` extensions) simply don't have async equivalents.

**Savepoints and nested transactions.** `async with db.begin_nested():` opens a `SAVEPOINT` inside the current transaction — if the nested block raises, only work since the savepoint rolls back, not the whole outer transaction. This is the tool for "try an operation that might fail, but keep the rest of the unit of work" — e.g., attempting several independent side-effects (like Surwill's best-effort shop-news fan-out in `_notify_shop_savers`, Lesson on notifications) where a failure in one shouldn't undo the primary write. Surwill's actual implementation there uses a full `try/except` with explicit `rollback()` rather than a savepoint because the fan-out runs in its own logical unit after the primary commit — but recognizing when a savepoint *would* be the right tool (partial rollback within one still-open transaction) is an interview-relevant distinction from a full rollback.

## Best practices

- **Default every `relationship()` to `lazy="raise"` on a new project.** Retrofitting it later is far more expensive than living with the verbosity from day one.
- **Extract primitive values from ORM objects into plain dicts/dataclasses before they cross a Session boundary** (end of request, background task handoff, cache write) — never hold onto an ORM instance and hope it's still attached later.
- **Use `Numeric`/`Decimal` for any value that represents money**, and cast to `str` (not `float`) at API boundaries to avoid reintroducing binary-float error in the client.
- **Put business-rule filters in the WHERE clause, not in Python after the fetch**, when the database can express them — it's both faster (less data over the wire) and the query itself becomes self-documenting.
- **Choose `JSON`/`JSONB` deliberately, not by default** — only when you need Postgres to see inside the blob (index, filter, or extract a key). Otherwise a text column with manual serialization is simpler and just as correct.
- **Let the database enforce referential integrity (`ondelete=`) even when the ORM also has cascade logic** — the DB constraint is the backstop that holds even when application code is bypassed.

## Common mistakes & gotchas

- **Assuming `.first()` means "at most one exists."** It silently discards extra rows. If your invariant is "exactly 0 or 1," use `scalar_one_or_none()` so a broken invariant surfaces as an exception, not a silent wrong answer.
- **Forgetting the soft-delete filter.** Every query against a `SoftDeleteMixin` table needs `deleted_at.is_(None)` by hand in Surwill's design — there's no default-manager magic hiding it for you (see Lesson 17 for how Django handles this differently, and the trade-off that implies).
- **Mixing `float` into a money code path "just for a quick calculation."** Even one `float(price_estimate)` intermediate step can reintroduce rounding error that a later `Decimal` comparison then fails on unpredictably.
- **Calling `session.commit()` and then continuing to use the same ORM objects assuming they're "done."** After commit, they're expired by default; the next access refetches. That's usually fine, but it's an extra round trip you may not have intended, and it's the first domino in a `DetachedInstanceError` if the Session closes before that access happens.
- **Adding a relationship without deciding its `ondelete` behavior**, leaving it at the (Postgres) default `NO ACTION`, which blocks deleting the parent unless children are handled first — a surprise that only shows up the first time someone tries to delete a row with dependents.

## Where AI helps (and where it hurts)

**Helps:** generating the boilerplate of a new model (columns, mixins, a first-draft `relationship()`) from a description of the domain; explaining an unfamiliar error message like `DetachedInstanceError` or `sqlalchemy.exc.InvalidRequestError: '...' is not available due to lazy='raise'`; suggesting which loading strategy fits a specific access pattern once you describe it. **Hurts:** letting an AI choose `ondelete` behavior or cascade rules without you supplying the actual business rule — "what should happen to X when Y is deleted" is domain knowledge, not something inferable from the schema alone, and a wrong guess here (e.g., `CASCADE` where `SET NULL` was intended) can silently delete data in a way that's hard to notice until it's already happened. Similarly, don't let AI pick `lazy=` defaults for you across a whole model file without understanding the access-pattern trade-off yourself — you need to be able to defend the choice in review and in an interview.

## Learn independently

- **Architecture Patterns with Python** (Percival & Gregory) — Chapter 2 ("Repository Pattern") and Chapter 6 ("Unit of Work Pattern") are the conceptual foundation for everything SQLAlchemy's `Session` implements; read these *before* the SQLAlchemy docs and the API will make far more sense.
- **SQLAlchemy 2.0 official documentation** — *ORM Querying Guide* (relationship loading techniques): https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html — read the "Selecting from LEFT OUTER JOIN" vs "IN clause" comparison at the top; it's the same trade-off explained above with more visual query-plan detail.
- **SQLAlchemy 2.0 official documentation** — *Cascades*: https://docs.sqlalchemy.org/en/20/orm/cascades.html — the full table of ORM-level cascade options (`save-update`, `delete`, `delete-orphan`, `merge`) and how they differ from DB-level `ondelete`.
- **Designing Data-Intensive Applications** (Kleppmann) — Chapter 7 ("Transactions") for the theory behind what "identity map + unit of work" is actually protecting you from at the concurrency level.
- **PostgreSQL documentation** — §8.1 Numeric Types (https://www.postgresql.org/docs/current/datatype-numeric.html) and §5.4.5 Foreign Key constraints (https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-FK) for the database-side half of everything in this lesson.

## Interview preparation

**Q: What's the difference between `lazy="select"`, `lazy="joined"`, and `lazy="selectin"`? When would you pick each?**
A: `select` (the default) issues a brand-new query the first time you touch the attribute — simplest, but causes N+1 if you access it in a loop over many parents. `joined` folds the related rows into the *original* query via a `LEFT OUTER JOIN` — one round trip total, but for one-to-many it duplicates the parent's columns across every joined child row on the wire, and it always pays that cost even when you don't end up needing the relationship. `selectin` runs a *second* query but batches it — `WHERE parent_id IN (...)` for every parent already loaded — so it's O(1) additional queries regardless of how many parents you loaded, at the cost of one extra round trip. In practice: `selectin` is the safe default for one-to-many; `joined` fits many-to-one/one-to-one where you almost always need the related row and it's small.

**Q: Why would a team deliberately choose `lazy="raise"` over just being careful about N+1?**
A: Because "being careful" doesn't scale across a team or across time — a relationship access that's fine today (3 rows in dev) becomes a production incident eight months later when a different engineer, in a different part of the codebase, accesses the same attribute inside a loop over a now-large table. `lazy="raise"` converts that class of bug from "silent until it's slow in production" into "a test fails the moment the code is written," because touching the relationship without having eager-loaded it raises immediately, in any environment, at any data size.

**Q: A colleague reports a `DetachedInstanceError` — walk me through how you'd debug it.**
A: It means code is accessing an attribute on an ORM object whose `Session` has already been closed (or the object was expunged), and the attribute wasn't already loaded into memory, so SQLAlchemy needs to issue a query to fetch it and has no connection to do so. I'd look for where the object crossed a boundary — end of a request context, handed to a background task, cached and read back later — and check whether the specific attribute being accessed was already loaded (e.g., via eager loading) *before* that boundary. The fix is almost always to extract the needed primitive values into a plain dict/dataclass while the Session is still open, rather than holding onto the live ORM object.

**Q: [System design / trade-off] Your team is split: half want to switch every relationship to `selectin` by default for "safety," half want `lazy="raise"` like Surwill does. How do you decide?**
A: It depends on the codebase's maturity and size. For a *new* project, `lazy="raise"` is close to free to adopt and pays for itself the first time it prevents an accidental N+1 from reaching production — the verbosity cost (always specifying `.options()`) is small and arguably a feature, since it documents exactly what each query fetches. For a *large, existing* codebase with hundreds of call sites already assuming implicit lazy loading, flipping to `raise` all at once would break all of them simultaneously — there, `selectin` as a safer *default* (still not free, but never silently N+1-inducing across a loop) with case-by-case overrides is the pragmatic migration path, potentially moving toward `raise` relationship-by-relationship as each one gets touched anyway.

**Q: Why does Surwill store `price_estimate` as `Numeric` and cast to `str` in the API response instead of returning a JSON number?**
A: Two separate concerns. `Numeric`/`Decimal` avoids binary floating-point rounding error inside the database and Python layers — essential for money, where compounding rounding errors are unacceptable. Casting to `str` at the API boundary avoids a *different* problem: JSON's number type is itself typically parsed into IEEE-754 double-precision float by JavaScript clients, which would silently reintroduce the exact precision loss `Decimal` was protecting against, the moment the value crosses into the frontend. Sending it as a string forces the client to parse it deliberately (or just display it, which is most of Surwill's use case) rather than accidentally doing float arithmetic on it.

## Exercises against the codebase

**Easy:** Find every `relationship()` declaration in the Surwill backend (`grep -rn "relationship(" app/`). For each, identify the `lazy=` setting and write one sentence for each explaining what would happen — in terms of queries fired — if you removed the `lazy="raise"` and let it default to `"select"`, in the specific service function that uses that relationship.

**Medium:** `_backfill_from_existing_item` reuses data from *any* item that matches the URL, regardless of which user saved it. Rewrite it (as a design sketch, not necessarily runnable code) to *also* prefer an item belonging to the *same user* first, falling back to any user's item second, while still respecting the existing quality filters (`title`, `price_estimate` non-null) and freshness ordering. Explain, in the query you write, whether this is one query or two, and why.

**Hard:** Surwill's `Notification.payload` is a `JSON` column queried via `Notification.payload[payload_key].astext == payload_value` in `_recent_exists` (Lesson on notifications). Write the Alembic migration (see Lesson 07's format) that would add a **GIN index** on this column to make that lookup efficient at scale, explain in a comment why a plain B-tree index wouldn't help here, and describe — from `EXPLAIN ANALYZE` behavior you'd expect, not by actually running it unless you have a local Postgres — how the query plan would change with the index in place versus without it.

## Key takeaways

- SQLAlchemy's `Session` is a Unit of Work *and* an Identity Map — understand both halves, not just "it manages connections."
- `relationship()` configures *how* a related row would be fetched; `lazy=` configures *when* — they're separate axes, and conflating them is the source of most confusion about "why did this query fire."
- `lazy="raise"` is Surwill's deliberate trade: pay verbosity at the query site, in exchange for making expensive-by-accident structurally impossible rather than just discouraged.
- Money is `Numeric`/`Decimal`, never `float`, all the way to the API boundary, where it becomes a `str` to avoid reintroducing float error in the client.
- `ondelete=` at the database level is the integrity backstop that holds even when the ORM is bypassed; ORM-level cascades are a convenience layered on top, not a replacement.
- Extract primitive values from ORM objects *before* the Session that loaded them might close — that discipline is what prevents `DetachedInstanceError`, not a `try/except` after the fact.
