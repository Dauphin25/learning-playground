# Lesson 03 — Routing, Requests & Responses

**Track:** FastAPI (Surwill) · **Time:** ~2h · **Prerequisites:** lessons 01–02
**Objectives:** master `APIRouter` composition, path/query/body parameter binding, `response_model` semantics, status codes, and the error-response vocabulary Surwill exposes.

---

## Why this matters

Routing is where FastAPI's core trick lives: **the function signature is the
contract**. Path types, request bodies, auth, response shape, and the OpenAPI
docs all derive from the same annotations. Mid-level interviews test whether you
know what the framework does *for* you (validation, serialization, docs) and
what it will never do for you (authorization, consistent error shapes).

## Theory

A FastAPI route is declared by decorating a function on an `APIRouter`. At
import time FastAPI inspects the signature and classifies every parameter:

- Appears in the path (`{board_id}`) → **path param**, converted+validated to the annotated type.
- Scalar type not in path (`int`, `str`, `bool`) → **query param**.
- Pydantic model → **request body** (at most implicit one; more via `Body()`).
- `Annotated[..., Depends(...)]` → **dependency** (lesson 05).
- `Request`, `Response`, `BackgroundTasks` → injected raw objects.

Validation failures short-circuit into a `422` with a machine-readable error
list before your code runs. `response_model` does the inverse on the way out:
the return value is *filtered and coerced* through the declared model — fields
not in the model are stripped, which is a security feature (an ORM object's
`password_hash` cannot leak through a response model that doesn't declare it).

## Official documentation

> "You can declare path 'parameters' or 'variables' with the same syntax used by
> Python format strings."
> — FastAPI docs, https://fastapi.tiangolo.com/tutorial/path-params/

The response-model chapter (https://fastapi.tiangolo.com/tutorial/response-model/)
documents the filtering behavior described above.

## The code, explained

### Composition root

All routers are stitched together in `app/main.py`:

```python
app.include_router(auth_router,     prefix="/api/v1/auth",     tags=["auth"])
app.include_router(users_router,    prefix="/api/v1/users",    tags=["users"])
app.include_router(boards_router,   prefix="/api/v1/boards",   tags=["boards"])
app.include_router(items_router,    prefix="/api/v1/items",    tags=["items"])
app.include_router(admin_router,    prefix="/api/v1",          tags=["admin"])
...
```

Each slice exports a bare `APIRouter()`; the *mount point* is decided centrally.
That keeps versioning (`/api/v1`) in exactly one file — an eventual `/api/v2`
is a second `include_router` block, not a refactor. `tags` group endpoints in
the Swagger UI.

### A read-heavy route with the full parameter menu

From `app/items/routes.py`:

```python
@router.post("/boards/{board_id}/items", response_model=ItemResponse, status_code=201,
             tags=["items"])
async def create_item(
    board_id: uuid.UUID,          # path param — 404s malformed UUIDs? No: 422
    body: ItemCreate,             # JSON body via Pydantic
    current_user_id: CurrentUserID,  # dependency (JWT auth)
    db: DBSession,                # dependency (AsyncSession)
) -> ItemResponse:
    item = await services.create_item(board_id, uuid.UUID(current_user_id), body, db)
    return ItemResponse.model_validate(item)
```

Details worth naming out loud:

- `board_id: uuid.UUID` — FastAPI parses and validates the path segment; a
  malformed UUID never reaches your code (automatic 422). Using UUIDs
  everywhere also kills enumeration attacks that sequential integer IDs invite.
- `status_code=201` — semantics matter: creation returns 201, deletes return
  204 with no body (`@router.delete("/{item_id}", status_code=204)` returns
  `None`).
- `ItemResponse.model_validate(item)` — explicit ORM→DTO conversion (Pydantic
  v2 `from_attributes`); the double safety with `response_model` costs little
  and keeps the return type honest for mypy.

### Errors as vocabulary, not improvisation

Surwill never raises `HTTPException(status_code=403, ...)` inline in services.
It subclasses once in `app/shared/exceptions.py`:

```python
class ForbiddenError(HTTPException):
    def __init__(self, detail: str = "Forbidden") -> None:
        super().__init__(status_code=status.HTTP_403_FORBIDDEN, detail=detail)

class ProfileIncompleteError(HTTPException):
    def __init__(self, detail: str = "Complete your profile before adding items.") -> None:
        super().__init__(
            status_code=status.HTTP_428_PRECONDITION_REQUIRED,
            detail={"code": "PROFILE_INCOMPLETE", "message": detail},
        )
```

Two lessons in ten lines. First, the **domain exception vocabulary**
(`NotFoundError`, `ForbiddenError`, `ConflictError`, `TooManyRequestsError`,
`UnprocessableError`) lets services express business outcomes while remaining
HTTP-renderable — the service layer stays readable and the mapping to status
codes lives in one file. Second, `ProfileIncompleteError` carries a
**machine-readable `code`** inside `detail`, so the frontend can react
programmatically (pop the profile form) instead of string-matching an English
sentence. The frontend's API client (`messageFromDetail` in `lib/api.ts`)
mirrors this contract.

Design note an interviewer may push on: making domain exceptions *subclass*
`HTTPException` couples services to HTTP. The alternative is pure domain
exceptions plus `@app.exception_handler` translators. Surwill chose the
pragmatic coupling; know the purist option exists and what it buys (reusing
services from a CLI/worker without HTTP semantics).

### The auth slice as a routing tour

`app/auth/routes.py` shows more shapes:

```python
@router.post("/register", response_model=TokenResponse)
async def register(body: RegisterRequest, db: DBSession) -> TokenResponse: ...

@router.post("/login", response_model=TokenResponse)
async def login(body: LoginRequest, request: Request, db: DBSession, redis: Redis) -> TokenResponse:
    await services.check_rate_limit(
        redis, f"login_rate:{_client_ip(request)}:{body.email}", limit=10, window_seconds=900
    )
    ...

@router.post("/logout", status_code=204)
async def logout(body: RefreshRequest, db: DBSession) -> None: ...
```

- Raw `Request` injected alongside Pydantic body — needed for the client IP
  (`X-Forwarded-For` first, `request.client.host` fallback).
- The 204 route returns `None` — FastAPI sends an empty body.
- Note what is *absent*: no `try/except` anywhere. Domain exceptions raised in
  `services` propagate and render themselves.

## How this is used in production

Real APIs live and die by contract stability: `response_model` + OpenAPI gives
you a generated, always-current contract that frontend teams and SDK generators
(e.g. openapi-typescript) consume. Enterprises wire the OpenAPI JSON into CI
diff checks ("did this PR break the contract?"). Error-shape consistency (the
`{"detail": {"code", "message"}}` convention) is what makes client error
handling sane across dozens of endpoints — every serious platform (Stripe is
the canonical example) publishes exactly such an error vocabulary with
machine-readable codes.

## Advanced corner

- **Route matching order** is declaration order within a router; `/items/stats`
  must be declared before `/items/{item_id}` or `"stats"` will be parsed as an
  ID and 422. Surwill's slices are small enough to dodge this; know the rule.
- **`response_model` costs a validation pass** on every response. For large list
  endpoints, that's real CPU. Escapes: `response_model=None` + manual dicts, or
  ORJSONResponse for serialization speed. Measure first.
- **`model_validate` vs returning ORM objects directly:** returning the ORM
  object works (FastAPI validates it through the response model), but explicit
  conversion pins where lazy-loading could fire. With `lazy="raise"`
  relationships (lesson 06), a response model that touches an unloaded
  relationship fails loudly in dev instead of emitting N+1 queries in prod —
  an underrated combination.
- **HEAD/OPTIONS** are auto-handled (OPTIONS by CORSMiddleware, HEAD by
  Starlette from GET routes) — don't write them.

## Best practices

- One prefix per version, applied centrally at `include_router`.
- Domain exceptions once, in `shared/exceptions.py`; machine-readable `code`
  fields for anything a client must branch on.
- 201 for create, 204+`None` for delete/logout, 409 for state conflicts,
  422 for semantic rejects, 428/`PROFILE_INCOMPLETE`-style codes for flow control.
- UUID path params everywhere — validation + anti-enumeration for free.

## Common mistakes & gotchas

- Declaring the more generic path first and shadowing the specific one.
- Returning dicts that "happen to match" the response model — until a rename
  silently drops a field. Use `model_validate`.
- Leaking ORM internals by *not* using a response model — the classic
  `password_hash` in JSON incident.
- Using 200 for everything; interviewers notice 201/204/409 discipline.

## Where AI helps (and hurts)

AI generates CRUD routes flawlessly from an example — Surwill's uniform slices
are ideal prompting material. It's weak at **status-code semantics** and at
keeping error shapes consistent across a codebase; those are review points.

## Learn independently

- FastAPI tutorial, "Path Parameters" through "Handling Errors":
  https://fastapi.tiangolo.com/tutorial/
- **Architecture Patterns with Python**, ch. 4 (the service-layer/API split shown here).
- RFC 9110 (HTTP Semantics) §15 — the status-code registry, actually worth skimming once.
- *Robust Python* (Viafore), ch. 1–3 on making types carry intent — the philosophy behind signature-as-contract.

## Interview Q&A

**Q: How does FastAPI decide what's a path param vs query vs body?**
A: "By signature inspection: names matching path template variables are path
params; remaining scalars are query params; Pydantic models become the JSON
body; `Depends`-annotated params invoke DI; raw `Request`/`Response` are
injected. Everything is validated before the handler runs, and a failure is an
automatic 422 listing each bad field."

**Q: What does `response_model` actually do?**
A: "Three things: documents the response in OpenAPI, validates/coerces the
return value, and — most importantly — *filters* it. Only declared fields
survive, so sensitive ORM attributes can't leak. It costs a validation pass,
which you might trade away on hot list endpoints."

**Q: How do you keep error responses consistent across a large API?**
A: "Define the error vocabulary once — in my project, HTTPException subclasses
like `ConflictError` and `ProfileIncompleteError` in `shared/exceptions.py`,
raised from the service layer. Anything the client needs to branch on carries a
machine-readable `code`, not just English text. The alternative — inline
HTTPExceptions everywhere — decays into inconsistent statuses within a month."

## Exercises against the codebase

- **Easy:** Hit `POST /api/v1/auth/register` with a bad email and a short
  password; read the 422 body and map each entry to the schema constraint.
- **Medium:** Add `GET /api/v1/items/{item_id}` returning `ItemResponse`, 404
  via `NotFoundError`, owner-only via the existing helpers. Route stays ≤6 lines.
- **Hard:** Add a `format=csv` query param to an existing list endpoint that
  switches the response to `text/csv` via a custom `Response` — learn what you
  lose (response_model, docs) and document the trade.

## Key takeaways

- The signature is the contract: params classified by inspection, 422s free.
- `response_model` documents, validates, and **filters** — a security control.
- Central router composition = one-file versioning; central exception vocabulary = consistent errors with machine-readable codes.
- Status-code discipline (201/204/409/428) is cheap senior-signal.
