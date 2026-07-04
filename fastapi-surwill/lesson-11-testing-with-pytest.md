# Lesson 11 — DEEP DIVE: Testing with pytest

**Track:** FastAPI (Surwill) · **Time:** ~3.5h · **Prerequisites:** lessons 05, 06, 09, 10
**Objectives:** master pytest for an async FastAPI app — fixtures, `conftest.py`, dependency overrides, `ASGITransport`, async-loop/pooling pitfalls, golden-file tests, mocking, and a real debugging war story — well enough to explain the whole strategy and every layer of the pyramid in the codebase.

---

## Why this matters

"How do you test this?" follows every system-design answer. A weak reply names a
framework; a strong one describes a *strategy* — what's unit vs integration,
where the boundaries and doubles are, how the DB is handled, how flaky async
tests are avoided. Surwill's suite (80 tests across auth, permissions,
reservations, and seven scraping files) is a compact, real illustration of all
of it, including a genuine async-pooling bug that was diagnosed and fixed.

## Theory

### pytest's model

pytest discovers `test_*` functions, injects **fixtures** by parameter name,
and reports with rich assert introspection (plain `assert a == b`, no
`assertEqual`). The pillars:

- **Fixtures** — setup/teardown as injectable, composable functions. A fixture
  yielding a value runs teardown after `yield`. **Scope** (`function` default,
  `session`, `module`) controls how often it runs.
- **`conftest.py`** — fixtures visible to every test in its directory tree, no
  import. The suite's shared harness lives here.
- **Parametrization** — `@pytest.mark.parametrize` runs one test body over many
  inputs (each a separate reported case).
- **Markers** — `@pytest.mark.asyncio`, custom tags, `skip`/`xfail`.

### The testing pyramid, seen in this repo

- **Unit (many, fast, no I/O):** the scraping engine and helpers — `extract()`
  on a fixed HTML string, `parse_price`, `assert_safe_url` on literal IPs,
  managed-API request builders. Pure functions, microseconds each.
- **Integration (fewer, real DB):** service tests — `reserve_item`,
  `can_view_board` — against a real Postgres `AsyncSession`.
- **API/E2E (fewest, full stack):** auth flows through `AsyncClient` +
  `ASGITransport`, exercising routing, DI, validation, and DB together.

Surwill leans unit-heavy exactly where logic is dense (scraping) and
integration where correctness is subtle (reservations, permissions). That
*shape* is the interview answer to "what's your testing strategy."

> "pytest: helps you write better programs … makes it easy to write small,
> readable tests, and can scale to support complex functional testing."
> — pytest docs, https://docs.pytest.org/

## The code, explained: `tests/conftest.py`

```python
TEST_DATABASE_URL = os.environ.get(
    "TEST_DATABASE_URL", "postgresql+asyncpg://surwill:surwill@localhost:5432/surwill_test"
)

# NullPool: each pytest test runs in its own event loop; pooled asyncpg
# connections must not be reused across loops.
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False, poolclass=NullPool)
TestSession = async_sessionmaker(test_engine, expire_on_commit=False)

@pytest_asyncio.fixture(scope="session")
async def setup_db():
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest_asyncio.fixture
async def db(setup_db):
    async with TestSession() as session:
        yield session
        await session.rollback()

@pytest_asyncio.fixture
async def redis_mock():
    mock = AsyncMock()
    mock.get = AsyncMock(return_value=None)
    mock.pipeline = MagicMock(return_value=AsyncMock())
    mock.setex = AsyncMock()
    return mock

@pytest_asyncio.fixture
async def client(db, redis_mock):
    app.dependency_overrides[get_session] = lambda: db
    app.dependency_overrides[get_redis]   = lambda: redis_mock
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

This 50-line file is the whole harness. Every design decision matters:

**`TEST_DATABASE_URL` from env with a localhost default.** The suite runs in two
places — locally against `localhost:5432`, and in CI against a service
container (`@db:5432`). Env-overridable config makes one conftest serve both;
the default keeps `pytest` frictionless on a dev machine. (Making this
env-overridable was part of the same change that fixed the pooling bug below.)

**Real Postgres, not SQLite.** The integration tests use an actual Postgres
because the code under test uses Postgres-only features — `SELECT … FOR UPDATE`
(reservations), `UUID` columns, `timestamptz`. Testing against SQLite would
pass while production breaks. This is a deliberate, defensible choice; the cost
is "you need a Postgres to run tests," which the Docker workflow provides.

**`ASGITransport` — HTTP tests with no socket.** `AsyncClient(transport=ASGITransport(app=app))`
routes httpx requests straight into the ASGI app *in-process* — real routing,
middleware, DI, validation, serialization, but no network, no running server.
Fast and deterministic. `base_url="http://test"` is a dummy authority for URL
construction.

**Dependency overrides = the seam.** Lesson 05's payoff cashed here: the
`client` fixture swaps `get_session` for the test's transactional `db` and
`get_redis` for an `AsyncMock`. No monkeypatching, no import surgery — the app
runs exactly as in production except the two injected edges. Teardown clears the
overrides so tests don't leak into each other.

**`redis_mock` as an `AsyncMock`.** Rate limiting, caching, and queueing touch
Redis; the API tests don't care about Redis behavior, so it's mocked to return
"nothing cached, nothing rate-limited." A test that *does* care (rate-limit
exceeded) overrides one method locally:

```python
redis_mock.get = AsyncMock(return_value="3")   # simulate 3 prior attempts → 429
```

That's the mocking sweet spot: default double in the fixture, per-test override
for the branch under test.

## War story: `InterfaceError: another operation is in progress`

This actually happened while building the email-auth test suite, and it's a
top-tier interview story because it's a *real* async-testing failure with a
non-obvious fix.

**Symptom.** The new auth tests failed en masse with
`asyncpg ... InterfaceError: cannot perform operation: another operation is in
progress` and `RuntimeError: ... got Future attached to a different loop`.

**Diagnosis.** pytest-asyncio can run each test in its **own event loop**. A
SQLAlchemy async engine with a normal connection **pool** caches asyncpg
connections; a connection created on test A's loop got handed to test B on a
*different* loop. asyncpg connections are bound to the loop that made them —
using one from another loop is the "operation in progress"/"different loop"
error. The bug wasn't in the tests; it was in the engine's pooling interacting
with per-test loops.

**Fix.**

```python
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False, poolclass=NullPool)
```

`NullPool` disables connection reuse — every checkout opens a fresh connection
on the *current* loop and closes it after. Slightly slower, completely correct.
(Also pinned `pytest-asyncio<0.24`, whose 1.x line changed loop/fixture
semantics and reintroduced the breakage.)

**Why it's a great story:** it shows you can read an async traceback, reason
about event-loop/connection affinity, and fix the *root cause* (pooling) instead
of the symptom (retrying, or downgrading blindly). Interviewers remember the
person who debugged the loop, not the person who used a bigger timeout.

## Test types in the suite

### Golden-file tests (the scraping engine's safety net)

```python
# tests/test_scraping_golden.py
_FIXTURES = Path(__file__).parent / "fixtures"

def test_zoommer_ge_product_extraction() -> None:
    html = (_FIXTURES / "zoommer_ge_product.html").read_text(encoding="utf-8")
    data = scraper.extract("https://zoommer.ge/…/xiaomi-tv-…-p50792", html)
    normalize_prices(data)
    assert data.get("title") == "Xiaomi TV S Pro Mini led 65 2026 Black"
    assert data.get("price") == "2099"
    assert data.get("original_price") == "2499"   # previousPrice from platform JSON
    assert data.get("in_stock") is True           # isInStock flag
    assert data.get("category") == "ტელევიზორები"
```

A **real store page saved to `tests/fixtures/`** (some are 100–200 KB of actual
HTML) is fed through the real extractor with assertions on every field. Why this
is the *only* sane way to test a 2600-line fuzzy extractor:

- **Deterministic** — no network, so it never flakes on a store being down or
  changing; the fixture is frozen HTML captured on a known date.
- **Regression lock** — any engine change that breaks zoommer extraction fails
  this test loudly. The extractor can be refactored fearlessly.
- **Executable spec** — "we extract these exact fields from this real page" is
  documented by a test that must stay true.
- **Extension workflow** (lesson 10) — new store = new fixture + new golden
  test; add adapter selectors only until it's green.

This is the concrete resolution of the tension in lesson 10: the engine is fuzzy
and un-unit-reasonable, so its correctness is pinned empirically against real
captured inputs.

### Pure unit tests (fast, no fixtures)

```python
# tests/test_scraping_security.py
@pytest.mark.parametrize("url", [
    "http://127.0.0.1/admin", "http://169.254.169.254/latest/meta-data/",
    "http://[::1]/", "http://0.0.0.0/",
])
async def test_blocks_private_and_loopback_ips(url):
    with pytest.raises(UnsafeURLError):
        await assert_safe_url(url)
```

Parametrization turns one body into many reported cases — each blocked IP is its
own pass/fail. The literal-IP cases need no DNS, so they're pure and instant.
`tests/test_scraping_managed.py` similarly tests the pure `_build_request` /
`_extract_html` functions with zero network — which is *why* those functions
were written pure in the first place (lesson 10). **Testability is a design
input, not an afterthought.**

### Integration tests (real DB, the critical invariants)

```python
# tests/reservations/test_reservations.py
async def test_double_reservation_raises_conflict(db: AsyncSession) -> None:
    owner  = await _create_user(db, "+995555002001", "owner2")
    giver1 = await _create_user(db, "+995555002002", "giver2")
    giver2 = await _create_user(db, "+995555002003", "giver3")
    board  = await _create_public_board(db, owner)
    item   = await _create_item(db, board)
    await reserve_item(item.id, giver1.id, ReservationCreate(), db)
    with pytest.raises(ConflictError):
        await reserve_item(item.id, giver2.id, ReservationCreate(), db)
```

These test the **business rules that matter most** (the file's own docstring
says so): an item can't be double-reserved, an owner can't reserve their own
item, permissions gate visibility. They call the *service function* directly
with a real session — no HTTP, because the rule lives in the service, not the
route. Small `_create_*` helpers build the object graph; testing at the service
layer (lesson 01) is what makes this concise.

### API tests (full stack via the client fixture)

```python
# tests/test_auth.py
async def test_password_reset_full_flow(client: AsyncClient, redis_mock):
    payload = _register_payload("reset.flow@example.com")
    register = await client.post("/api/v1/auth/register", json=payload)
    old_refresh = register.json()["refresh_token"]
    requested = await client.post("/api/v1/auth/password-reset/request", json={"email": payload["email"]})
    token = requested.json()["dev_token"]
    confirmed = await client.post("/api/v1/auth/password-reset/confirm",
                                  json={"token": token, "new_password": "brand-new-pass1"})
    assert confirmed.status_code == 200
    # old password dead, new works, pre-reset refresh revoked, token single-use:
    assert (await client.post("/api/v1/auth/login", json={"email": payload["email"], "password": payload["password"]})).status_code == 422
    assert (await client.post("/api/v1/auth/login", json={"email": payload["email"], "password": "brand-new-pass1"})).status_code == 200
    assert (await client.post("/api/v1/auth/refresh", json={"refresh_token": old_refresh})).status_code == 422
```

One test asserts an entire security contract end to end (register → request →
confirm → old-pw-fails → new-pw-works → old-sessions-revoked → token-single-use).
It exercises routing, validation, DI, the service, and the DB together — the
things unit tests can't see. Note it relies on the **dev-only echo of the reset
token** (`dev_token`, suppressed in production) to avoid reading email — a
deliberate testability affordance.

## Running the suite (this repo's reality)

Tests run **inside the api container** — `app/main.py` hardcodes
`StaticFiles(directory="/app/static")` (a container path), so a host run fails
at import. The workflow:

```bash
# fresh test DB (drop + create are separate psql statements)
docker compose exec -T db psql -U surwill -c "DROP DATABASE IF EXISTS surwill_test"
docker compose exec -T db psql -U surwill -c "CREATE DATABASE surwill_test"
# run against the db service, not localhost
docker compose exec -T -e TEST_DATABASE_URL=postgresql+asyncpg://surwill:surwill@db:5432/surwill_test \
    api python -m pytest tests -q
# → 80 passed
```

CI (lesson 12) does the same with Postgres/Redis **service containers** and
`TEST_DATABASE_URL` pointed at them — the env-overridable URL is what lets one
conftest serve laptop and CI unchanged.

## How this is used in production

Serious teams gate merges on the pyramid: fast unit tests on every push, the
integration/API layer against ephemeral DB/Redis service containers, coverage
tracked (pytest-cov) but read with judgment (100% coverage of trivial getters
proves nothing; the reservation race test is worth a thousand). Golden/snapshot
testing is standard wherever output is complex and fuzzy — scrapers, template
renderers, serializers, ML feature pipelines. `ASGITransport`-style in-process
API testing is the FastAPI/Starlette norm because it's fast enough to run
hundreds per CI minute. Test data via factories (factory-boy is in Surwill's
dev deps) replaces hand-built graphs as suites grow.

## Advanced corner

- **Event-loop/pool affinity** (the war story) generalizes: any pooled async
  resource (DB, Redis, http client) shared across per-test loops will bite you.
  `NullPool` for the *test* engine is the standard fix; alternatively pin a
  single session-scoped loop. Know both.
- **Transaction-rollback isolation.** Surwill's `db` fixture rolls back after
  each test; a stronger pattern wraps each test in a SAVEPOINT/nested
  transaction rolled back in teardown, so tests never see each other's writes
  even without recreating the schema. Trade-off: speed vs isolation strictness.
- **Test DB per worker for `pytest-xdist`.** Parallelizing means each worker
  needs its own database (`surwill_test_gw0`, …) or they clobber each other —
  the `TEST_DATABASE_URL` env indirection is exactly the hook for that.
- **Mock at the boundary, not the internals.** Surwill mocks Redis (an external
  edge) but uses a *real* DB (core behavior). Over-mocking (patching your own
  services) produces tests that pass while the system is broken — the classic
  "tests are green, prod is down."
- **What's deliberately NOT tested:** live network scrapes. They'd be flaky and
  slow; golden files capture the input once and test the deterministic core.
  Being able to say "I test the extractor deterministically and leave live
  fetching to monitoring" is mature.
- **`pytest-asyncio` mode.** The repo uses `asyncio_mode = "auto"`
  (pyproject) so `async def test_*` needs no per-test marker. The 1.x pin exists
  because that line changed fixture-loop semantics — a dependency-hygiene point.

## Best practices

- Pyramid: many pure unit tests where logic is dense; fewer integration tests
  for subtle correctness; a thin API layer for wiring.
- One `conftest.py` harness; dependency overrides as the seam; clear overrides
  in teardown.
- Real DB for DB-dependent logic; mock only true external edges.
- Golden files for fuzzy output; parametrize for input families.
- Make config env-overridable so laptop == CI.

## Common mistakes & gotchas

- **Pooled async engine + per-test loops** → `another operation is in progress`.
  Use `NullPool` (the war story).
- Testing DB logic on SQLite because it's easy → passes while `FOR UPDATE`/UUID
  behavior diverges in prod.
- Forgetting `dependency_overrides.clear()` → state bleeds; test order matters.
- Over-mocking your own code → green suite, broken system.
- Asserting on live scrapes → perennial flakiness; freeze a fixture instead.
- Running tests on the host here → import fails on the hardcoded `/app/static`.

## Where AI helps (and hurts)

AI writes fixture scaffolding, parametrized cases, and golden-test bodies
quickly, and it's good at generating edge-case input lists (all the private-IP
ranges for the SSRF test). It's weak precisely where this lesson is deep:
**async-loop/pooling bugs** (it'll suggest a timeout bump, not `NullPool`) and
**mock boundaries** (it happily patches your own services into meaninglessness).
Let it draft; you decide the seam and diagnose the flake.

## Learn independently

- pytest docs — fixtures, parametrize, conftest:
  https://docs.pytest.org/en/stable/how-to/fixtures.html
- pytest-asyncio docs (modes, loop scope): https://pytest-asyncio.readthedocs.io/
- httpx `ASGITransport` + FastAPI "Testing":
  https://fastapi.tiangolo.com/advanced/async-tests/
- **Architecture Patterns with Python** — the "testing" threads throughout
  (service-layer tests, fakes vs mocks).
- **Robust Python** (Viafore) — ch. on testing types/contracts; and Percival's
  *Test-Driven Development with Python* for the discipline.

## Interview Q&A

**Q: Describe your testing strategy.**
A: "A pyramid matched to where risk lives. The scraping engine is dense fuzzy
logic, so it's mostly pure unit tests plus golden-file tests — real store HTML
frozen as fixtures, run through the real extractor with field assertions, so
it's deterministic and locks against regressions. The subtle correctness rules —
double-reservation, permissions — are integration tests calling the service
function against a real Postgres, because they use `FOR UPDATE` and UUIDs that
SQLite wouldn't model. A thin top layer tests full auth flows through httpx's
ASGITransport, which runs the whole app in-process. Redis is mocked because it's
an external edge; the database is real because it's core behavior."

**Q: How do you test HTTP endpoints without running a server?**
A: "httpx `AsyncClient` with `ASGITransport(app=app)` — requests go straight
into the ASGI app in-process, so routing, middleware, dependency injection,
validation, and serialization all run, but there's no socket. In the fixture I
override the session and Redis dependencies with a test transaction and an
AsyncMock, and clear the overrides in teardown. It's fast enough to run the
whole API suite in seconds."

**Q: Tell me about a hard bug you debugged.** *(use the war story)*
A: "Building the auth test suite, every async DB test suddenly failed with
asyncpg's 'another operation is in progress' and 'Future attached to a different
loop.' The cause was subtle: pytest-asyncio ran tests on separate event loops,
but the SQLAlchemy engine's connection pool cached asyncpg connections, and an
asyncpg connection is bound to the loop that created it — so a connection made
on one test's loop got reused on another's. The fix was `NullPool` on the test
engine so every test opens a fresh connection on its own loop, plus pinning
pytest-asyncio below 1.0 where the loop semantics changed. It's the difference
between fixing the symptom and understanding event-loop/connection affinity."

**Q: Why real Postgres in tests instead of SQLite?**
A: "The code under test uses Postgres-specific behavior — `SELECT … FOR UPDATE`
row locking for reservations, native UUID and timestamptz columns. On SQLite
those tests would pass against semantics production doesn't have. The cost is
needing a Postgres to run tests, which Docker and CI service containers provide;
an env-overridable `TEST_DATABASE_URL` lets the same conftest target my laptop
or the CI container."

## Exercises against the codebase

- **Easy:** Run `pytest tests/test_scraping_security.py -v` and read how one
  parametrized function becomes N reported cases; add IPv6 unique-local
  (`fc00::/7`) to the blocked list and its case.
- **Medium:** Add an API test for the login rate limiter: set
  `redis_mock.get` to return the limit and assert a 429 (mirrors the reset-flow
  test's structure).
- **Hard:** Reproduce the war story deliberately — swap `NullPool` for the
  default pool in a scratch conftest, watch the async tests break, read the
  traceback, then restore it. Write a three-sentence root-cause note. Bonus:
  make the suite `pytest-xdist`-parallel with a per-worker test DB.

## Key takeaways

- Match the pyramid to risk: pure/golden for the fuzzy scraper, real-DB
  integration for invariants, thin ASGITransport layer for wiring.
- `conftest.py` + dependency overrides is the harness; mock external edges
  (Redis), keep the core real (Postgres).
- Golden-file tests are the only sane way to pin a 2600-line fuzzy extractor —
  frozen real HTML, deterministic assertions.
- The `NullPool` war story = event-loop/connection affinity; fix root cause, not symptom.
- Testability is a design input: pure functions (`_build_request`), service-layer logic, and dev-only token echoes all exist partly *to be tested*.
