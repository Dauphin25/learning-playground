# Lesson 05 — Dependency Injection

**Track:** FastAPI (Surwill) · **Time:** ~2h · **Prerequisites:** lessons 02–04
**Objectives:** read `app/deps.py` like a native; understand `Depends`, `Annotated` aliases, generator dependencies, dependency chaining for auth, and why DI is what makes the test suite possible.

---

## Why this matters

Dependency injection is FastAPI's most distinctive design and the highest-yield
topic in FastAPI interviews. It answers, in one mechanism: how does a route get
a DB session, how is auth enforced without repeating code, how do tests swap
infrastructure, and how are resources cleaned up per-request. If you can
whiteboard `deps.py` from memory, you can answer a whole question category.

## Theory

A **dependency** is any callable FastAPI can resolve before your handler runs.
Declaring `db: Annotated[AsyncSession, Depends(get_session)]` tells the
framework: call `get_session()`, await it if needed, and pass the result. The
resolver:

- **Chains** — dependencies can depend on dependencies; FastAPI builds and
  resolves the DAG.
- **Caches per request** — the same dependency appearing twice in one request
  graph runs once (`use_cache=True` default).
- **Supports generators** — a `yield` dependency runs its post-`yield` code
  after the response, giving per-request setup/teardown (this is how DB
  sessions close).
- **Is overridable** — `app.dependency_overrides[real] = fake` swaps any node
  in the graph, which is the official testing mechanism.

> "'Dependency Injection' means, in programming, that there is a way for your
> code (in this case, your *path operation functions*) to declare things that it
> requires to work and use: 'dependencies'."
> — FastAPI docs, https://fastapi.tiangolo.com/tutorial/dependencies/

`Annotated[T, Depends(f)]` (PEP 593) attaches the DI metadata to the type
without disturbing type checkers: mypy sees `T`, FastAPI sees `Depends(f)`.

## The code, explained: `app/deps.py`

The whole DI surface of the application is one 85-line file. The exported
vocabulary is at the bottom:

```python
DBSession             = Annotated[AsyncSession, Depends(get_session)]
Redis                 = Annotated[aioredis.Redis, Depends(get_redis)]
CurrentUserID         = Annotated[str, Depends(get_current_user_id)]
OptionalCurrentUserID = Annotated[str | None, Depends(get_optional_user_id)]
AdminUser             = Annotated["User", Depends(require_admin)]
```

These aliases are why route signatures across twelve slices read identically:
`db: DBSession, current_user_id: CurrentUserID`. One import, no `Depends`
noise at call sites, and refactoring the wiring means editing one file.

### The session dependency (generator style)

```python
# app/db.py
async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

Per request: session created → handed to the handler → after the response, the
`async with` closes it (returning the connection to the pool) even if the
handler raised. Resource lifecycle in three lines, no middleware, no globals.

### Auth as a dependency chain

```python
bearer_scheme = HTTPBearer()

async def get_current_user_id(
    credentials: Annotated[HTTPAuthorizationCredentials, Depends(bearer_scheme)],
) -> str:
    token = credentials.credentials
    payload = decode_access_token(token)
    if payload is None:
        raise HTTPException(status_code=401, detail="Invalid or expired token",
                            headers={"WWW-Authenticate": "Bearer"})
    user_id: str | None = payload.get("sub")
    if user_id is None:
        raise HTTPException(status_code=401, detail="Invalid token payload")
    return user_id
```

Chain: `HTTPBearer` (extracts+validates the `Authorization: Bearer …` header,
401 if absent) → `get_current_user_id` (verifies signature+expiry+type claim,
returns the `sub`). Because `HTTPBearer` is a security scheme, Swagger UI gets a
lock icon and a token box for free.

Notice what this dependency **doesn't** do: hit the database. Authentication
costs one HMAC check per request; the user row is loaded only where routes need
it. That's a deliberate latency choice — and its trade-off (revocation lag until
token expiry, 15 minutes here) is a favorite interview follow-up.

### Optional auth — same mechanism, softer edge

```python
optional_bearer_scheme = HTTPBearer(auto_error=False)

async def get_optional_user_id(credentials = Depends(optional_bearer_scheme)) -> str | None:
    if credentials is None:
        return None
    payload = decode_access_token(credentials.credentials)
    return payload.get("sub") if payload else None
```

Public wishlist pages are visible to anonymous users but show more to logged-in
viewers (private items you were granted). `auto_error=False` turns "missing
header → 401" into "missing header → None", and the *service* decides what the
viewer may see (`list_user_public_items(username, viewer_id, db)`). Authn in
the dependency, authz in the service — clean split.

### Authorization tier: `require_admin`

```python
async def require_admin(
    current_user_id: Annotated[str, Depends(get_current_user_id)],
    db: Annotated[AsyncSession, Depends(get_session)],
) -> "User":
    user = await db.scalar(select(User).where(User.id == uuid.UUID(current_user_id)))
    if not user or not user.is_admin:
        raise HTTPException(status_code=403, detail="Admin access required")
    return user
```

This *composes* the auth chain and adds a DB check — three levels deep
(`HTTPBearer` → `get_current_user_id` → `require_admin`), and thanks to
per-request caching, a route using both `CurrentUserID` and `AdminUser` runs the
token decode once. Every admin route is protected by its **signature**:
`admin: AdminUser` — grep-ably, unforgettably.

### Process-wide singletons behind dependencies

```python
_redis_pool: aioredis.Redis | None = None

async def get_redis() -> aioredis.Redis:
    global _redis_pool
    if _redis_pool is None:
        if not settings.is_production and settings.REDIS_URL == "fakeredis://":
            import fakeredis
            _redis_pool = fakeredis.FakeAsyncRedis(decode_responses=True)
        else:
            _redis_pool = aioredis.from_url(settings.REDIS_URL, decode_responses=True)
    return _redis_pool
```

The dependency *interface* is per-request; the *resource* behind it is a lazy
process singleton (a connection pool must not be built per request). Note the
dev nicety: `REDIS_URL=fakeredis://` swaps in an in-memory Redis so the backend
runs without any Redis at all. Configuration-driven test doubles, in prod code,
gated to non-production — pragmatic and safe.

### The payoff: overrides in tests (`tests/conftest.py`)

```python
app.dependency_overrides[get_session] = lambda: db          # test transaction
app.dependency_overrides[get_redis]   = lambda: redis_mock  # AsyncMock
```

The entire app now runs against a test database session and a mock Redis — no
monkeypatching, no import-order games. Lesson 11 builds on this.

## How this is used in production

DI is how real FastAPI apps stay testable at scale: teams define exactly such a
`deps.py` vocabulary, then services receive everything explicitly. Multi-tenant
systems resolve the tenant from the JWT/host in a dependency and inject a
tenant-scoped session. Rate limiting, feature flags, locale, pagination params —
all standard dependency material. The override mechanism doubles as a chaos
tool: inject a latency-adding session wrapper in staging and watch your p99.

## Advanced corner

- **`use_cache=False`** forces re-execution of a dependency within a request —
  needed when a dependency is intentionally effectful per call site (rare;
  reach for it consciously).
- **`yield` + exception flow:** code after `yield` runs after the response is
  sent; wrapping the yield in `try/finally` is how you guarantee cleanup. An
  exception raised *by the handler* propagates through the generator, so a
  session dependency could `rollback()` on error — Surwill relies on
  `async with`'s close semantics instead, with services committing explicitly.
- **Router- and app-level dependencies:** `APIRouter(dependencies=[Depends(require_admin)])`
  applies a guard to every route in the router — Surwill guards per-route via
  `AdminUser`; the router-level form is the stronger default for a large admin
  surface (can't forget one).
- **Sync dependencies in async apps** run in the threadpool — same rules as
  sync endpoints (lesson 02), same saturation risk.

## Best practices

- One `deps.py`; export `Annotated` aliases, not raw `Depends` at call sites.
- Auth chain returns the *minimum* (user id string); load full rows only where needed.
- Authn in dependencies, authz decisions that need context in services (with a
  central helper like `can_view_board`).
- Expensive clients: lazy singleton behind a dependency, never per-request.

## Common mistakes & gotchas

- Creating an engine/pool inside a dependency body — one pool per request,
  connection exhaustion within minutes under load.
- Doing DB work in auth for every request when the token already carries what
  you need — you pay a query per request across the whole API.
- Forgetting `app.dependency_overrides.clear()` between tests — state leaks
  make test order matter (Surwill's `client` fixture clears in teardown).
- Using `Depends()` in non-route helper functions and wondering why nothing
  injects — DI happens only where FastAPI calls you.

## Where AI helps (and hurts)

AI writes dependency plumbing well once shown your aliases. Watch it on
**caching semantics** (it will re-create pools per request) and on **401 vs
403** (authentication vs authorization) — it conflates them constantly.

## Learn independently

- FastAPI dependencies chapter, all sub-pages (classes as dependencies,
  sub-dependencies, dependencies with yield):
  https://fastapi.tiangolo.com/tutorial/dependencies/
- Security chapter (OAuth2/JWT flows): https://fastapi.tiangolo.com/tutorial/security/
- **Architecture Patterns with Python**, ch. 13 — DI and composition roots
  beyond any framework.
- PEP 593 (`Annotated`) — the ten-minute read that demystifies the syntax.

## Interview Q&A

**Q: Explain FastAPI's dependency injection and why it beats globals.**
A: "Handlers declare requirements in their signature; the framework resolves a
DAG of callables per request, caching repeated nodes, running post-`yield`
teardown after the response. Versus globals: lifecycle is per-request and leak-
proof, the graph is explicit and type-checked, and any node can be overridden
in tests — we swap the DB session and Redis with two dict entries, no patching."

**Q: How does auth work per request in your project?**
A: "A three-level chain: `HTTPBearer` parses the header, `get_current_user_id`
verifies the JWT (signature, expiry, token-type claim) and returns the subject;
routes that need role checks use `require_admin`, which composes the chain and
loads the user. Auth costs one HMAC verification, no DB hit for ordinary
routes. The trade-off is revocation lag bounded by the 15-minute access-token
TTL — refresh tokens are the revocable layer."

**Q: A route needs the current user in three different dependencies. How many
times does the token get decoded?**
A: "Once — dependency results are cached per request by default, keyed on the
callable. `use_cache=False` opts out when you genuinely need re-execution."

## Exercises against the codebase

- **Easy:** Add a `RequestID = Annotated[str, Depends(...)]` dependency reading
  the `X-Request-ID` header (set by the logging middleware) and use it in one route.
- **Medium:** Build `PaginationParams` as a *class dependency* (`limit` capped
  at 100, `offset ≥ 0`) and refactor one list endpoint to use it.
- **Hard:** Write a `require_board_owner(board_id, user_id, db)` dependency that
  404s/403s appropriately, and refactor `items/routes.py` to use it — observe
  how much service code it absorbs, then argue whether that's an improvement
  (there's a real argument against: authz drifting out of services).

## Key takeaways

- `deps.py` is the app's DI vocabulary: five `Annotated` aliases cover 90% of routes.
- Chains: HTTPBearer → user id → admin; cached per request; JWT auth = zero DB hits.
- Generator dependencies give per-request lifecycle; singletons live behind them.
- `dependency_overrides` is the testing superpower — the whole suite hangs off it.
