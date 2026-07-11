# Lesson 17 — From Django to FastAPI: A Translation Guide

**Track:** FastAPI (Surwill) · **Lesson:** 17 of 17 · **Estimated time:** 3 hours
**Prerequisites:** Working Django/Django REST Framework knowledge (assumed, not taught here); Lessons 01–14 of this track
**Learning objectives:**
- Map every major Django/DRF concept to its FastAPI+SQLAlchemy equivalent, and understand *why* the equivalent looks different, not just what to type instead
- Recognize which Django conveniences FastAPI deliberately does not provide, and what that trade-off buys you
- Convert a Django-shaped instinct (fat models, signals, middleware-heavy auth) into idiomatic FastAPI/Surwill-shaped code
- Know precisely what's lost and what's gained moving from sync WSGI to async ASGI
- Walk into an interview and explain *why* a team would choose FastAPI over Django for a given problem — and when Django is still the better call

---

## Why this matters

You already know Django. That's an asset, not a liability — most of what makes someone dangerous with FastAPI is the ability to *translate* existing instincts rather than learn web development from zero. But translation done carelessly produces "Django code wearing a FastAPI costume": fat views that do too much, an ORM used the sync way inside an async framework, a re-invented admin panel, signals recreated as ad-hoc event buses. This lesson exists to make the translation *deliberate* — for every Django concept, you'll see the direct FastAPI/SQLAlchemy equivalent from Surwill's real code, and, just as importantly, you'll see **which Django conveniences have no equivalent on purpose**, because FastAPI's whole design philosophy is different from Django's.

The one-sentence version of that philosophy difference, which will anchor everything below: **Django is a batteries-included, opinionated, synchronous-by-default full-stack framework built around the ORM and the admin; FastAPI is a thin, async-native, unopinionated web layer built around type hints and dependency injection, that assumes you'll bring your own ORM, your own admin (or none), your own everything else.** Every difference in this lesson traces back to that one sentence.

## Theory

### The big picture: what Django gives you "for free" that FastAPI doesn't

| Concern | Django | FastAPI |
|---|---|---|
| ORM | Built in (`django.db.models`), sync by default (async support since 4.1, still maturing) | Not included — Surwill brings SQLAlchemy 2.0, async-native |
| Admin panel | Built in, auto-generated from models | Not included — Surwill has no admin UI in the backend; `/vakontroleb` is a hand-built Next.js page hitting normal API routes |
| Migrations | Built in (`makemigrations`/`migrate`) | Not included — Surwill brings Alembic (Lesson 07) |
| Forms & validation | `django.forms` / DRF Serializers | Not included as a separate concept — Pydantic models *are* both the request/response schema and the validator (Lesson 04) |
| Auth | Built in (`django.contrib.auth`), session-based by default, DRF adds token/JWT patterns | Not included — Surwill hand-rolls JWT issuance/refresh (Lesson 08) |
| Templates | Built in (Django Template Language) | Not applicable — FastAPI is API-only here; Surwill's frontend is a separate Next.js app (this track never touches HTML rendering) |
| URL routing | `urls.py`, string-based path converters | Decorator-based path operations (`@router.get("/items/{id}")`), Starlette's `{id}` converters |
| Request/response shape | Implicit (views return whatever; DRF serializers add structure) | Explicit and enforced — a path operation's `response_model` is part of its type signature |
| Background jobs | Not included — virtually every Django shop adds Celery | Not included either — Surwill adds **arq** (Lesson 09), Redis-based, async-native, much lighter than Celery |
| Concurrency model | WSGI — synchronous, one worker thread/process per request by default | ASGI — asynchronous, one event loop handling many concurrent requests per worker |

The pattern across every row: Django decided *for* you; FastAPI makes you decide, but gives you first-class primitives (dependency injection, Pydantic, ASGI) to build the decision on cleanly. Neither is "better" in the abstract — it's a trade between velocity-via-convention (Django) and precision-via-explicitness (FastAPI), and the right call depends on the project.

### Models: Django ORM vs. SQLAlchemy — same job, different philosophy

```python
# Django — a model IS the table AND (often) business logic
class Item(models.Model):
    title = models.CharField(max_length=300)
    price_estimate = models.DecimalField(max_digits=10, decimal_places=2, null=True)
    board = models.ForeignKey("Board", on_delete=models.CASCADE, related_name="items")
    in_stock = models.BooleanField(null=True)

    class Meta:
        ordering = ["-created_at"]

    def is_on_sale(self) -> bool:  # fat-model business logic, common Django idiom
        return self.original_price and self.original_price > self.price_estimate
```

```python
# SQLAlchemy — app/items/models.py (Surwill), a model IS the table, nothing else
class Item(Base, UUIDMixin, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "items"
    title: Mapped[str] = mapped_column(String(300), nullable=False)
    price_estimate: Mapped[Decimal | None] = mapped_column(Numeric(10, 2), nullable=True)
    board_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("boards.id", ondelete="CASCADE"), nullable=False, index=True
    )
    board: Mapped["Board"] = relationship(back_populates="items", lazy="raise")
    in_stock: Mapped[bool | None] = mapped_column(Boolean, nullable=True)
```

Structurally these look almost the same — both declare columns as class attributes with type/constraint info. The real difference is **where behavior lives**. Django's "fat model, thin view" convention actively encourages methods like `is_on_sale()` directly on the model — the model is simultaneously the schema *and* the place business logic accumulates, because `django.db.models.Model` is designed to be subclassed with your own methods, and the ORM's "active record"-adjacent style (an instance knows how to `.save()` itself) makes this feel natural.

Surwill's SQLAlchemy models are deliberately **inert** — no business-logic methods, ever. `discount_pct` isn't a method on `Item`; it's computed in the *service layer* or the *response schema* from raw column values. This is a direct consequence of the layered architecture Lesson 01 introduced: routes → schemas → services → models, where models are the bottom, dumbest layer on purpose, so that logic is testable in isolation from the database and easy to find (it's always in `services.py`, never scattered across model methods). This is closer to the **Data Mapper** pattern (SQLAlchemy's own description of itself) than Django's **Active Record** pattern — the model doesn't know how to persist itself; the `Session` does that, from the outside.

**Query syntax, side by side:**

```python
# Django: manager-based, chainable, lazy QuerySet
items = Item.objects.filter(board__owner=user, is_archived=False).order_by("-created_at")[:20]
item = Item.objects.get(id=item_id)              # raises DoesNotExist / MultipleObjectsReturned
item, created = Item.objects.get_or_create(...)  # a Django-only convenience with no direct SA equivalent
```

```python
# SQLAlchemy 2.0: explicit select(), executed against a Session
result = await db.execute(
    select(Item)
    .join(Board, Item.board_id == Board.id)
    .where(Board.owner_id == user_id, Item.is_archived.is_(False))
    .order_by(Item.created_at.desc())
    .limit(20)
)
items = result.scalars().all()

item = await db.get(Item, item_id)  # returns None, doesn't raise
# or, when "must exist or it's a real error":
item = (await db.execute(select(Item).where(Item.id == item_id))).scalar_one_or_none()
```

The Django version reads slightly shorter — that's the "batteries included" trade paying off. But notice what SQLAlchemy's version makes *explicit* that Django's makes *implicit*: the `join`. Django's `board__owner=user` double-underscore syntax **silently generates a JOIN across the relationship** — convenient, but it means you can accidentally generate a large, expensive join without ever typing the word "join," and query-plan surprises are harder to spot by reading the code. SQLAlchemy makes you write the join, which is more typing but means the code and the query plan are never a surprise to each other.

**The single biggest gotcha for a Django developer:** Django QuerySets are lazy and **auto-execute** the moment you iterate, slice for display, or call `len()` — you rarely think about "when does the query actually run." SQLAlchemy's `select()` is a query *description* that does nothing until you `await session.execute(...)` it — there is no ambient "just iterate it and it'll query" behavior, because in an async world, *something* has to `await`, and Python has no way to make plain iteration (`for x in queryset`) awaitable in the transparent way Django gets away with synchronously. This is not a small stylistic difference — it's a direct consequence of async.

### Views/Serializers vs. Path operations/Pydantic — collapsing two layers into one

Django REST Framework is the fairer comparison to FastAPI than plain Django (which returns HTML, not JSON, by default). DRF separates a `ViewSet`/`APIView` (routing + request handling) from a `Serializer` (validation + shape) from `urls.py` (routing table). FastAPI collapses routing, validation, and (via type hints) documentation into one function:

```python
# Django REST Framework
class ItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = Item
        fields = ["id", "title", "price_estimate", "in_stock"]

class ItemViewSet(viewsets.ModelViewSet):
    serializer_class = ItemSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Item.objects.filter(board__owner=self.request.user)

# urls.py
router.register("items", ItemViewSet)
```

```python
# FastAPI — app/users/routes.py (Surwill), the equivalent in one function
@router.get("/me/items", response_model=list[WishItemResponse], tags=["wish-list"])
async def get_my_items(
    current_user_id: CurrentUserID,
    db: DBSession,
    limit: int | None = Query(default=None, ge=1, le=200),
) -> list[WishItemResponse]:
    items = await item_services.list_user_items(uuid.UUID(current_user_id), db, limit=limit)
    return [WishItemResponse.from_item(i) for i in items]
```

`WishItemResponse` (a Pydantic model, Lesson 04) plays the role DRF's `ItemSerializer` plays — declaring the response shape — but it's also the *validation* layer if it were a request body, and it's also what generates the OpenAPI schema, automatically, from the same type hints. DRF needs a `Serializer` for validation, a `Serializer` (often the same one) for output shape, and separate OpenAPI tooling (`drf-spectacular` or similar) bolted on to generate docs from those serializers. FastAPI gets all three from one Pydantic model, because Pydantic models are genuinely just Python type definitions with runtime validation attached — there's no separate "schema description" artifact to keep in sync.

`current_user_id: CurrentUserID` is FastAPI's dependency injection (Lesson 05) doing the job of DRF's `permission_classes = [IsAuthenticated]` — but notice the difference in *how* it fails and *what* it looks like in the function signature. DRF's permission check happens in a place you can't see from reading the view method — it's declared as class metadata, enforced by the framework before your method even runs. FastAPI's `Depends()`-based auth is a parameter — it's *part of the function's type signature*, so reading the function signature alone tells you this endpoint requires auth, without needing to know DRF's permission-class convention exists.

### Auth: Django sessions vs. hand-rolled JWT

Django's default auth is **session-based**: login sets a signed cookie referencing a server-side session record; every request, Django's `AuthenticationMiddleware` looks up that session and populates `request.user`. DRF layers token/JWT auth on top for APIs (via `djangorestframework-simplejwt` or similar), but the *session* model is Django's native center of gravity — `request.user` "just exists" because middleware put it there before your view ran.

FastAPI has no built-in auth system at all — Surwill hand-rolls JWT access + refresh token issuance and rotation (Lesson 08), and "who is the current user" is answered by a `Depends()`-based dependency (`CurrentUserID`) that decodes and validates the bearer token *per request*, not by middleware populating a global. This means: no server-side session storage (stateless, horizontally scalable without sticky sessions or a shared session store) — but also no built-in mechanism for "log this user out everywhere right now" the way invalidating a Django session record does; Surwill's refresh-token rotation (single-use, tracked) is the mechanism that fills that gap, and it's application code, not a framework feature.

### The admin panel — the single biggest "batteries" Django has that FastAPI doesn't

Django's auto-generated admin (`django.contrib.admin`) is arguably its most famous feature: register a model, get a working CRUD UI for free, with search, filters, and inline editing. FastAPI has **nothing like this**, on purpose — it's a web *API* framework, not a full-stack one. Surwill's answer is `/vakontroleb` — a hand-built page in the **Next.js frontend**, calling ordinary `/api/v1/admin/*` FastAPI routes that have no special framework support; they're regular path operations, gated by an `is_admin` check in the same `Depends()` style as any other auth. This is strictly more work than Django's admin, and it's a real cost of choosing FastAPI — teams that lean heavily on Django admin for internal tooling should weigh this deliberately, not discover it as a surprise mid-project.

### Migrations: `makemigrations`/`migrate` vs. Alembic

Conceptually identical — both diff your models against the database schema and generate a versioned migration script. Mechanically: Django's migration system is tightly coupled to Django's own ORM and auto-detects most changes with very little configuration. Alembic (Lesson 07) is a separate library, decoupled from SQLAlchemy by design, which means more manual specification (you often hand-write or hand-adjust the autogenerated migration, especially for data migrations or Postgres-specific DDL) but also means it works identically whether your app is FastAPI, Flask, or a bare script — it's not tied to one web framework's release cycle.

### Signals vs. explicit service calls

Django's `post_save`, `pre_delete`, etc. signals let far-away code react to a model change without the code that triggered the change knowing about it — e.g., a signal handler that sends a welcome email whenever a `User` is created, registered in a completely different app. Surwill has **no signal-equivalent system** — when `create_user_item` needs to also check whether it should notify shop-savers, that's an **explicit function call** in the service (`_notify_shop_savers`, see Lesson on notifications), visible in the same file, not a handler registered elsewhere that you'd have to know to go looking for. This is a deliberate trade: signals reduce coupling in the code graph but make control flow harder to trace (a `grep` for who calls a function finds every caller; a `grep` for a signal doesn't reliably find every *handler* without knowing Django's signal-connection conventions) — Surwill chooses traceable explicitness over decoupled convenience, consistent with `lazy="raise"` (Lesson 15) and the layered architecture's whole philosophy.

### Concurrency: WSGI/sync vs. ASGI/async — the deepest difference

This is the one difference that isn't really "Django vs. FastAPI" so much as "sync-Python-web vs. async-Python-web," and it reshapes everything else on this page.

Classic Django (WSGI) handles one request per worker thread/process at a time, synchronously — if a view blocks on a slow database query or an HTTP call to another service, that *worker* is stuck for the duration; you scale by adding more worker processes (gunicorn with N workers), each with its own memory footprint and its own DB connection pool slice. FastAPI (ASGI, via uvicorn) runs an event loop per worker that can juggle *many* concurrent requests on one thread, because `async def` handlers `await` at I/O boundaries (DB queries, HTTP calls) and hand control back to the loop instead of blocking it — Surwill's `_persist_item_refresh` (Lesson 09) fires up to 5 concurrent scrapes (`asyncio.Semaphore(5)`) *inside a single worker*, something a sync Django view has no native way to express without reaching for threads or a task queue.

The cost: every I/O-touching call in the request path must be genuinely async (an async driver like `asyncpg`, `httpx` in async mode, `redis.asyncio`) — mixing in a blocking, synchronous call (a sync `requests.get()`, a sync DB driver) inside an `async def` path operation **blocks the entire event loop**, stalling every *other* concurrent request that worker is handling, not just the one that made the blocking call. This is the single most dangerous class of bug when moving from Django's mental model to FastAPI's: "it's just a function call" stops being true the moment you're inside an event loop, and every dependency in the call chain has to honor that.

Django added `async def` view support (3.1+) and async ORM methods (4.1+, still catching up to full feature parity with the sync ORM as of recent releases), so this line is blurring — but Django's *ecosystem* (middleware, many third-party packages, the admin) is still overwhelmingly sync-first, so "Django, but async" in practice means constantly checking whether the piece you're reaching for actually supports it. FastAPI was async-first from its 2018 debut, so its ecosystem (Starlette underneath, most popular integrations) assumes async as the default, not a bolt-on.

## Official documentation quotes

> "FastAPI is a modern, fast (high-performance), web framework for building APIs with Python based on standard Python type hints." — FastAPI documentation, homepage, https://fastapi.tiangolo.com/

> "Django follows the 'Don't Repeat Yourself' (DRY) principle... Django's philosophy is 'batteries included' — it ships with an ORM, admin interface, authentication system, and more, out of the box." — Django documentation, *Design philosophies*, https://docs.djangoproject.com/en/stable/misc/design-philosophies/

> "ASGI (Asynchronous Server Gateway Interface) is a spiritual successor to WSGI, intended to provide a standard interface between async-capable Python web servers, frameworks, and applications." — ASGI specification, https://asgi.readthedocs.io/en/latest/introduction.html

> "SQLAlchemy is most famous for its object-relational mapper (ORM)... [it] presents a method of associating user-defined Python classes with database tables, and instances of those classes (objects) with rows in their corresponding tables." — SQLAlchemy documentation, *ORM Quick Start*, https://docs.sqlalchemy.org/en/20/orm/quickstart.html

## The code, explained

The comparisons throughout **Theory** above *are* the code-explained section for this lesson — each Django/DRF snippet is paired directly with its real Surwill equivalent. One more, worth calling out on its own: settings.

```python
# Django — settings.py, module-level constants, often with manual os.environ.get(...) calls
DEBUG = os.environ.get("DEBUG", "False") == "True"
DATABASES = {"default": {"ENGINE": "django.db.backends.postgresql", "NAME": os.environ["DB_NAME"], ...}}
SECRET_KEY = os.environ["SECRET_KEY"]  # crashes at import time if missing — arguably a feature
```

```python
# FastAPI — app/config.py (Surwill), Pydantic Settings: typed, validated at startup
class Settings(BaseSettings):
    APP_ENV: str = "development"
    DATABASE_URL: str
    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15

    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()  # raises a clear Pydantic ValidationError, listing every missing/invalid var, if config is bad
```

Both fail fast on missing required config — Django via a raw `KeyError` on `os.environ["SECRET_KEY"]`, Pydantic Settings via a structured `ValidationError` that lists *every* problem at once (not just the first missing var it happens to hit), with type coercion and validation built in (`ACCESS_TOKEN_EXPIRE_MINUTES: int` will reject a non-numeric env var with a clear error, where Django's raw `os.environ.get(...)` would silently hand you a string until something downstream broke confusingly).

## How this is used in production

- **Companies that pick Django** tend to be building products where the admin panel, the built-in auth/permissions system, and "one blessed way to do most things" save real time — content-heavy sites, internal tools, MVPs where a small team needs to move fast without inventing infrastructure. Instagram famously ran (and still runs significant parts of) its backend on Django, leaning heavily on its ORM and admin at massive scale, proving the "batteries included" model scales when the batteries are used well.
- **Companies that pick FastAPI** tend to be building API-first products, often with a separate frontend (exactly Surwill's shape: FastAPI backend + Next.js frontend, no server-rendered templates), microservices where async I/O-bound workloads (calling other services, scraping, streaming) dominate, or ML/data-serving APIs where FastAPI's Pydantic-driven validation and automatic OpenAPI docs are a strong fit for a contract many other services/clients depend on. Netflix and Uber have both published about using FastAPI for internal ML-serving and tooling APIs, specifically citing async performance and automatic docs as draws.
- **Hybrid reality**: plenty of production systems use *both* — a Django monolith for the admin-heavy, content-management side of a product, and a separate FastAPI service for a specific high-throughput or async-heavy API surface (webhooks, real-time features, ML inference) that would fight Django's sync-first grain. Recognizing "which parts of this system actually need FastAPI's strengths" rather than treating it as an all-or-nothing framework choice is itself a senior-level architectural instinct.
- **Migration path**: teams migrating *from* Django *to* FastAPI (a common scenario, and worth naming explicitly since it's likely your actual situation) typically keep the Django app as the admin/back-office system and stand up FastAPI as a new, separate service for a specific new API surface — a full rewrite is rarely justified purely on framework preference; it's justified when the concurrency model (async I/O) or the API-contract discipline (Pydantic + OpenAPI) solves a real, current pain point.

## Advanced corner

**"Fat model, thin view" (Django) vs. "thin everything, fat service" (Surwill/FastAPI) — this is a *design choice*, not a framework mandate.** Nothing stops you from writing business logic directly in a FastAPI path operation function, or from writing a thin-model Django app with a dedicated service layer (many mature Django codebases evolve toward exactly that once fat models get unwieldy). The frameworks nudge you in different directions by convention and by what's *easy* — Django's Active Record style makes `item.is_on_sale()` feel natural because the object already knows how to save itself, so "it also knows how to answer a question about itself" doesn't feel like a stretch. SQLAlchemy's Data Mapper style keeps the model and the persistence machinery separate, which makes "the model is just data, logic lives elsewhere" feel equally natural. Know that this is convention, not enforcement, so you're not surprised when you meet a Django codebase with a rich service layer or a FastAPI codebase with logic-heavy Pydantic validators.

**DRF's `ModelSerializer` auto-derivation vs. Pydantic's explicitness — a real trade-off, not just syntax.** `ModelSerializer` can infer an entire serializer from a model's fields with `fields = "__all__"` or a short list — extremely fast to write, but it means the API's shape is *implicitly* coupled to the database schema unless you're disciplined about overriding fields. `WishItemResponse` in Surwill is a hand-written Pydantic model, entirely decoupled from `Item`'s columns — it can rename fields (`price_estimate` → `price`), compute derived ones (`is_claimed`), and omit internal-only columns, all without any risk of a new database column accidentally leaking into the API response just because someone forgot to exclude it. This is the API-contract-as-source-of-truth philosophy: the response schema is deliberately authored, not derived, so it can never silently drift from what the database happens to contain.

**Why "just add `async` to your Django views" doesn't get you FastAPI's concurrency for free.** Async Django views help when the *view itself* is the bottleneck and it's genuinely awaiting I/O — but Django's ORM reaching full async parity is recent, much of the *middleware stack* (session handling, CSRF, many third-party packages) is still sync and gets wrapped in a thread-pool bridge (`sync_to_async`) rather than natively async, which reintroduces thread-pool contention exactly where you were trying to avoid it. FastAPI's advantage isn't "the word async" — it's that essentially the *entire* stack, from Starlette's routing up through `asyncpg`, is async-native with no sync bridge in the hot path, so the concurrency benefit is real all the way down, not just at the view layer.

## Best practices

- **Don't recreate Django's admin as a goal in itself.** Build the minimum internal tooling the team actually needs (Surwill's `/vakontroleb` is intentionally small), rather than trying to replicate Django admin's full feature set in a hand-rolled frontend — that's a losing time investment.
- **Treat Pydantic response models as the API contract, hand-authored** — resist the urge to auto-generate them from SQLAlchemy models 1:1; the decoupling is a feature (Lesson 04 goes deep on this).
- **Push logic out of path operations and into services**, mirroring what disciplined Django teams eventually do with fat models anyway — FastAPI just makes this the path of least resistance from day one.
- **Audit every dependency in an async request path for blocking calls** — a single sync driver or `requests.get()` call silently poisons the concurrency benefit you're paying async's complexity cost for.
- **Use Alembic's autogenerate as a starting draft, not a final answer** — review every migration, the way you'd review a Django migration, but expect to hand-edit more often since Alembic's detection is less automatic than Django's.

## Common mistakes & gotchas

- **Writing sync-style, blocking code inside `async def` handlers** because it "looks the same as Django" — this is the single most common and most damaging mistake Django developers make moving to FastAPI; it doesn't error, it just silently serializes all concurrent requests on that worker.
- **Expecting `request.user` to "just be there."** FastAPI has no ambient current-user global — every path operation that needs it must declare it as a dependency parameter, every time. This is more typing, and it's also why you can never *forget* whether an endpoint requires auth by glancing at its signature.
- **Reaching for a Django-style signal system** to decouple side effects, and building an ad-hoc, harder-to-trace equivalent instead of just calling the function directly. Surwill's explicit service calls exist because "grep for callers" should always find every consumer of a change.
- **Assuming Alembic autogenerate catches everything Django's `makemigrations` would.** It's weaker at detecting certain changes (e.g., some constraint or index-only changes) and won't touch anything it can't diff from the model metadata versus the live schema — always review the generated script.
- **Under-specifying Pydantic response models** ("just return the ORM object and let FastAPI serialize it") — without an explicit `response_model`, you risk leaking internal fields (password hashes, soft-delete timestamps) that a Django `ModelSerializer` would have required you to explicitly list, but that FastAPI won't stop you from returning by accident if you skip the schema.

## Where AI helps (and where it hurts)

**Helps:** mechanically translating a known Django pattern into FastAPI syntax once you've decided the equivalent — "here's my DRF serializer, write the Pydantic equivalent" is a great AI task, low-risk, easy to verify. Also useful for explaining an unfamiliar SQLAlchemy error by comparing it to the nearest Django ORM concept, which is exactly the kind of bridging this lesson does by hand. **Hurts:** letting AI decide *architectural* translations for you — whether logic belongs in a service vs. a path operation, whether a given endpoint needs a signal-equivalent side effect or should call it explicitly, whether an admin feature is worth building at all. These are judgment calls informed by your team's size and the project's actual needs, not something inferable from "how Django does it," and an AI asked to "convert my Django app to FastAPI" will happily produce fat-path-operation, signal-mimicking code that reproduces Django's shape without its coherence, because it's pattern-matching syntax, not re-deriving the design philosophy.

## Learn independently

- **Architecture Patterns with Python** (Percival & Gregory) — read this *specifically* as a Django developer; Chapter 1 explicitly contrasts "Active Record" (Django's style) against "Data Mapper" (SQLAlchemy's style) and makes the trade-offs precise in a way that generalizes far beyond either framework.
- **FastAPI official documentation** — the *Full Stack FastAPI Template* discussion and the *Alternatives* page (https://fastapi.tiangolo.com/alternatives/) is Sebastián Ramírez's own account of what FastAPI took and didn't take from Django REST Framework, Flask, and others — genuinely useful primary-source context for this lesson's whole premise.
- **Django documentation** — *Design philosophies* (https://docs.djangoproject.com/en/stable/misc/design-philosophies/) is short and worth re-reading now that you're comparing it against a framework with the opposite philosophy — it'll sharpen exactly what you're trading away.
- **SQLAlchemy documentation** — *ORM Quick Start* and *Migrating from 1.x* (if you've seen older SQLAlchemy) both help calibrate how 2.0's `select()`-style API (used throughout Surwill) differs from both Django's QuerySet API and SQLAlchemy's own older `Query` API.
- **Two Scoops of Django** (Greenfeld & Roy) — even though it's Django-specific, its chapters on "fat models, thin views" articulate the philosophy Surwill's service layer is reacting *against*; understanding what you're moving away from sharpens why the new shape looks the way it does.

## Interview preparation

**Q: You know Django well. Why would you choose FastAPI for a new project instead?**
A: It depends on the shape of the project. If it's API-first — serving a separate frontend or other services, with no need for a content-management admin — FastAPI's async-native concurrency, automatic OpenAPI docs from type hints, and Pydantic-enforced request/response contracts are a strong fit, especially if the workload is I/O-bound (calling other APIs, scraping, streaming) where Django's sync-first ecosystem fights you. If the project needs a fast-to-build admin interface, session-based auth, and "one blessed way to do most things" for a small team moving fast — Django's batteries typically win on velocity. I wouldn't choose FastAPI just because it's newer; I'd choose it because the project's actual concurrency and API-contract needs match what it's built for.

**Q: What's the single biggest mental-model shift moving from Django views to FastAPI path operations?**
A: That `async def` isn't just syntax — it changes what "safe" code looks like. In Django, calling a slow, blocking function inside a view just makes that one request slow. In FastAPI, calling a *synchronous, blocking* function inside an `async def` path operation blocks the entire event loop, stalling every other concurrent request that worker is currently handling. The fix isn't "add async to the function" — it's ensuring every I/O touchpoint in the call chain (database driver, HTTP client, cache client) is genuinely async, all the way down.

**Q: How does dependency injection in FastAPI (`Depends()`) compare to what Django gives you for something like authentication or permissions?**
A: Django's auth typically lives in middleware — `AuthenticationMiddleware` runs before your view and populates `request.user` as an ambient global your view can just read; permission checks in DRF are declared as class-level metadata (`permission_classes`) enforced by the framework before your method body runs. FastAPI's `Depends()` makes both of these an explicit *parameter* in the path operation's signature — you can see, just from reading the function definition, exactly what it depends on (current user, DB session, rate limiter, whatever), and FastAPI resolves that dependency graph per request, including sharing a dependency's result across multiple `Depends()` that need it in the same request. It trades Django's "less to type, but you have to know the middleware/class-attribute conventions exist" for "more explicit, but self-documenting from the function signature alone."

**Q: [System design / trade-off] Your company runs a mature Django monolith with a heavily-used admin panel. A new feature needs to consume a slow, async-heavy third-party API and fan out to hundreds of concurrent sub-requests. How would you approach this without a full rewrite?**
A: I wouldn't rewrite the monolith. I'd stand up a small, separate FastAPI service specifically for this new feature's API surface — it's the workload FastAPI's concurrency model is built for (many concurrent I/O-bound calls on one worker), and it can be deployed and scaled independently of the Django app. The Django monolith keeps owning the admin, the primary data model, and anything session/content-heavy. The two services would communicate over a well-defined API contract (which FastAPI's Pydantic + OpenAPI setup makes easy to keep honest), and depending on data ownership, either the FastAPI service reads from the same database (careful: two ORMs, two migration systems, needs discipline about who owns schema changes) or, more cleanly, it owns its own data and the two talk over HTTP/events. I'd push hard for the "separate service, clear contract, no shared database" version unless there's a strong reason not to, because two ORMs fighting over one schema is a maintenance trap.

## Exercises against the codebase

**Easy:** Pick any one Surwill route (e.g., `app/users/routes.py::get_my_items`) and rewrite it, on paper, as a Django REST Framework `APIView` + `Serializer` pair. Note every place you had to make a decision FastAPI's version made for you via type hints (or vice versa).

**Medium:** Surwill's `_notify_shop_savers` (called explicitly from `create` in `app/featured/services.py`) is the kind of side effect Django developers often reach for a `post_save` signal to implement. Write the Django signal-handler version of this feature (model, signal registration, handler function) and then write two sentences comparing how you'd *discover* this side effect exists in each version, six months later, as a new engineer reading the codebase for the first time.

**Hard:** Surwill's `Item` model has no admin UI backed by framework magic — `/vakontroleb` is hand-built. Design (routes + a rough Pydantic schema, not full implementation) the minimum FastAPI backend surface you'd need to support a *generic*, Django-admin-like list/edit/delete UI for **any** model in this app, without hand-writing a bespoke route per model. What would you have to give up (compared to Django admin) to keep this generic, and where would you draw the line and just hand-write routes instead?

## Key takeaways

- Every FastAPI/Surwill pattern in this lesson traces back to one philosophy difference: Django decides for you (batteries included, opinionated); FastAPI hands you sharp primitives (type hints, dependency injection, Pydantic) and expects you to decide.
- SQLAlchemy models are inert data mappers by convention in Surwill (Data Mapper style); Django models are commonly "fat" (Active Record style) — neither is enforced by the framework, but each framework's ergonomics nudge you toward its own default.
- Pydantic models collapse what DRF splits across Serializers, validators, and separately-maintained OpenAPI tooling into one type-hinted class.
- The sync-to-async shift (WSGI→ASGI) is the deepest difference — it's not a syntax change, it's a change in what "a blocking call" costs the rest of the system.
- FastAPI has no admin, no signals, no session-based auth out of the box — these aren't oversights, they're the "thin, unopinionated layer" trade-off, and Surwill's `/vakontroleb`, explicit service calls, and hand-rolled JWT are what filling those gaps deliberately looks like.
- Choosing between Django and FastAPI (or running both, for different parts of one system) is a real architectural decision with a right answer *for a given workload* — being able to argue both sides convincingly is exactly what a senior interview question in this space is testing for.
