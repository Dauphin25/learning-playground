# Lesson 02 — ASGI, uvicorn & the Application Object

**Track:** FastAPI (Surwill) · **Time:** ~2h · **Prerequisites:** lesson 01; basic `async/await`
**Objectives:** explain what actually runs when a request arrives; understand `app/main.py` completely — lifespan, middleware order, mounting, and why the health check is written the way it is.

---

## Why this matters

"FastAPI" is the top of a stack: **uvicorn** (server) speaks HTTP and translates
it into **ASGI** (protocol), **Starlette** (toolkit) does routing/middleware, and
FastAPI adds validation and DI on top. Interviewers probe this layering with
questions like "what is ASGI?" or "what does `--workers 2` actually do?" — and
the difference between a script-kiddie answer and an engineer's answer is
knowing which layer does what.

## Theory

### ASGI in one breath

WSGI (the old standard) is a synchronous function call: `app(environ,
start_response)` — one thread per in-flight request. That model cannot express
long-lived connections (WebSockets) or interleaved I/O. ASGI replaces it with an
async callable:

```python
async def app(scope, receive, send): ...
```

`scope` is a dict describing the connection (type, path, headers); `receive` and
`send` are async channels for events (request body chunks in, response chunks
out). Because the callable is async, **one process interleaves thousands of
in-flight requests on one event loop** — whenever a handler awaits the database,
the loop runs someone else's handler.

> "Uvicorn is an ASGI web server implementation for Python."
> — uvicorn docs, https://www.uvicorn.org/

> "Starlette is a lightweight ASGI framework/toolkit, which is ideal for building
> async web services in Python."
> — Starlette docs, https://www.starlette.io/

The critical operational consequence: **the event loop is a shared resource.**
One accidental blocking call (`time.sleep`, `smtplib` without a thread, a
CPU-heavy parse) freezes *every* request in that worker. Surwill's email module
shows the discipline — `smtplib` is blocking, so it runs via
`asyncio.to_thread(_send_sync, msg)` (`app/shared/email.py`).

### Processes vs. the event loop

`uvicorn --workers 2` forks two *processes*, each with its own event loop, GIL,
connection pools, and in-process caches. Requests are distributed by the kernel
(SO_REUSEPORT-style socket sharing). That means:

- In-process state (like Surwill's Google JWKS cache in `auth/services.py`) is
  **per worker** — fine for caches, fatal if you meant it as shared state.
- The database pool config (`pool_size=10, max_overflow=20`) multiplies by the
  worker count. Two workers → up to 60 Postgres connections. This is the classic
  production surprise.

## The code, explained: `app/main.py` top to bottom

```python
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s %(message)s")
_request_logger = logging.getLogger("surwill.request")

if settings.SENTRY_DSN:
    sentry_sdk.init(dsn=settings.SENTRY_DSN, environment=settings.APP_ENV)
```

Module level runs **once per process at import** — logging config and Sentry
init belong here, not in a request path.

### Lifespan

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    if settings.DATABASE_URL.startswith("sqlite"):
        await init_db()
    yield
```

The lifespan context manager brackets the application's lifetime: code before
`yield` runs at startup (before the first request), code after runs at
shutdown. Surwill uses it minimally — `create_all` for the SQLite dev mode,
because real deployments migrate with Alembic instead. In bigger apps this is
where you'd warm caches or open client pools. The old `@app.on_event("startup")`
API is deprecated in favor of exactly this pattern.

### The application object and environment-dependent docs

```python
app = FastAPI(
    lifespan=lifespan,
    title="Surwill API",
    version="1.0.0",
    docs_url=None if settings.is_production else "/api/docs",
    redoc_url=None if settings.is_production else "/api/redoc",
    openapi_url=None if settings.is_production else "/api/openapi.json",
)
```

Swagger UI is generated from the same type hints that validate requests — free
in dev, and deliberately disabled in production (smaller attack surface; the
schema enumerates every endpoint for an attacker).

### Middleware — and why order matters

```python
app.add_middleware(CORSMiddleware, allow_origins=settings.CORS_ORIGINS,
                   allow_credentials=True, allow_methods=["*"], allow_headers=["*"])
app.add_middleware(GZipMiddleware, minimum_size=500)

@app.middleware("http")
async def add_public_cache_headers(request, call_next): ...
@app.middleware("http")
async def add_security_headers(request, call_next): ...
@app.middleware("http")
async def request_logging(request, call_next): ...
```

Middleware wraps the app like an onion: **the last middleware added is the
outermost** — it sees the request first and the response last. Each
`@app.middleware("http")` function receives `call_next`, awaits it to get the
response from the inner layers, then may mutate it. Surwill's stack, outside-in:
request logging (times *everything*, stamps `X-Request-ID`) → security headers
(`nosniff`, `X-Frame-Options`, HSTS in prod) → public cache headers
(`Cache-Control: public, max-age=300` for anonymous-cacheable GETs like the
landing content — this lets a CDN absorb homepage traffic) → gzip → CORS →
router.

Two production details worth quoting in an interview:

- The cache middleware **excludes `/admin/` paths and non-200 responses** — a
  cached admin response or a cached error is a security/correctness bug.
- The logger **skips `/api/health`** — orchestrator polling would otherwise be
  most of your log volume.

### CORS, precisely

The browser (not the server) enforces CORS: JS on `https://surwill.ge` may only
read responses from the API if the API's `Access-Control-Allow-Origin` says so.
`allow_origins` comes from settings (`["http://localhost:3000"]` in dev, the
real domain in prod), and `allow_credentials=True` matters because the wildcard
`*` origin is forbidden when credentials are allowed — one reason the origins
are an explicit list.

### Static files and the health check

```python
app.mount("/static", StaticFiles(directory="/app/static"), name="static")
```

`mount` grafts a whole ASGI sub-application at a path prefix. Note the
container-absolute path — this is why the test suite must run inside Docker (a
gotcha lesson 11 returns to).

```python
@app.get("/api/health")
async def health() -> JSONResponse:
    checks = {}
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception:
        checks["database"] = "error"
    try:
        redis = await get_redis()
        await redis.ping()
        checks["redis"] = "ok"
    except Exception:
        checks["redis"] = "error"
    healthy = all(v == "ok" for v in checks.values())
    return JSONResponse(status_code=200 if healthy else 503, content={...})
```

A health endpoint that returns `{"status": "ok"}` unconditionally is a *liveness*
check pretending to be a *readiness* check. This one actually exercises both
dependencies and returns 503 when degraded, so Docker's healthcheck (and later
Railway) can restart or route around a broken instance. Interview soundbite:
"my health check tells the truth."

### The production command

```dockerfile
CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2 --proxy-headers --forwarded-allow-ips '*'"]
```

Migrations run before the server starts (safe for one replica; a multi-replica
deploy moves this to a release phase — a trade-off to volunteer in interviews).
`--proxy-headers` makes uvicorn trust `X-Forwarded-For`/`-Proto` from the
reverse proxy so `request.client` and scheme are real — which Surwill's login
rate limiter depends on for the client IP.

## How this is used in production

Startups run uvicorn directly (as here) behind a platform proxy; enterprises
often wrap it in gunicorn (`gunicorn -k uvicorn.workers.UvicornWorker`) for its
battle-tested process management (worker recycling, graceful reloads). Both are
one flag apart in behavior. Serious deployments add: structured JSON logs keyed
by the request ID (Surwill logs `rid=` for exactly this), OpenTelemetry traces
through the middleware, and readiness vs. liveness endpoints split apart so a
degraded dependency drains traffic without triggering restarts.

## Advanced corner

- **BaseHTTPMiddleware's cost.** `@app.middleware("http")` sugar wraps
  Starlette's `BaseHTTPMiddleware`, which runs the inner app in a separate anyio
  task and streams the response through a memory channel. It's convenient but
  adds overhead per layer and historically interacted badly with streaming
  responses and context propagation. Pure ASGI middleware (a class with
  `__call__(scope, receive, send)`) avoids that; know that the trade exists.
- **Where `def` endpoints go.** FastAPI runs synchronous (`def`) endpoints in a
  threadpool (anyio, default ~40 threads) so they don't block the loop. That
  saves you from accidental blocking — until the threadpool saturates. Surwill
  is `async def` everywhere and pushes blocking work to `asyncio.to_thread`
  explicitly.
- **Graceful shutdown.** On SIGTERM uvicorn stops accepting, waits for in-flight
  requests, then runs lifespan-shutdown. The arq worker logs the same discipline
  (`shutdown on SIGTERM ◆ 5 jobs complete`). Coordinated drains are what make
  zero-downtime deploys possible.

## Best practices

- Everything environment-dependent flows from `settings`; `main.py` contains no literals that differ between dev and prod.
- Health checks must exercise dependencies and are excluded from request logs.
- Add security headers centrally in middleware, not per-route.
- Keep module level cheap: config, clients, wiring — never I/O that can fail without context.

## Common mistakes & gotchas

- Blocking the loop (`requests`, `time.sleep`, bcrypt without a thread) — the
  whole worker stalls. Grep your own projects for `import requests` inside
  `async def`.
- Assuming middleware order is the order written — it's the reverse.
- `allow_origins=["*"]` with `allow_credentials=True` — silently broken per spec.
- Forgetting `--proxy-headers`, then "rate limiting by IP" limits your load balancer.

## Where AI helps (and hurts)

AI writes middleware boilerplate well. It routinely gets **middleware order**
and **event-loop blocking** wrong — review those two by hand, always.

## Learn independently

- ASGI spec (readable in an evening): https://asgi.readthedocs.io/en/latest/
- uvicorn deployment docs: https://www.uvicorn.org/deployment/
- Starlette middleware docs: https://www.starlette.io/middleware/
- **Fluent Python, 2nd ed.**, ch. 21 — the asyncio mental model.
- **High Performance Python** (Gorelick & Ozsvald), ch. 8 "Asynchronous I/O" — why the event loop wins for I/O-bound servers.

## Interview Q&A

**Q: What's the difference between WSGI and ASGI?**
A: "WSGI is a synchronous callable — one request occupies one thread until done,
and long-lived connections don't fit the model. ASGI is an async callable
receiving a connection scope plus receive/send channels, so one event loop
interleaves thousands of I/O-bound requests and can express WebSockets. FastAPI
is an ASGI framework; uvicorn is the ASGI server in front of it."

**Q: Your API is 'slow under load' but CPU is idle. First suspicion?**
A: "Something is blocking the event loop or exhausting a pool. I'd look for
synchronous I/O in async handlers, then check DB pool saturation — with
`pool_size=10, max_overflow=20` per worker, 30 concurrent slow queries queue
everyone else. The request-duration logging middleware gives me the p99 to
confirm, and the request ID correlates a slow request to its logs."

**Q: What runs when uvicorn starts with `--workers 2`?**
A: "Two OS processes, each importing the app fresh — so module-level code,
Sentry init, and lifespan run per worker, and each has its own event loop, DB
pool, and in-memory caches. Shared state must live in Redis or Postgres."

## Exercises against the codebase

- **Easy:** Add a temporary middleware that injects header `X-Lesson: 02`; prove
  with curl where it sits in the onion by logging before/after `call_next`.
- **Medium:** Break the health check on purpose (stop Redis; `docker compose stop
  redis`) and watch `/api/health` flip to 503 while the container goes unhealthy.
- **Hard:** Replace the `request_logging` decorator middleware with a pure ASGI
  middleware class of identical behavior. Measure overhead difference with `ab`
  or `hey` (expect small but real).

## Key takeaways

- uvicorn (server) → ASGI (contract) → Starlette (toolkit) → FastAPI (validation+DI).
- One event loop per worker process; blocking it stalls everyone; `--workers` multiplies pools and caches.
- Middleware wraps in reverse order of addition; Surwill's onion is logging → security → cache → gzip → CORS.
- Lifespan replaces startup/shutdown events; health checks must tell the truth.
