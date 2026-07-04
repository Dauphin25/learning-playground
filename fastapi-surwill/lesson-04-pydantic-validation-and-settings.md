# Lesson 04 — Pydantic: Validation, Schemas & Settings

**Track:** FastAPI (Surwill) · **Time:** ~2h · **Prerequisites:** lesson 03
**Objectives:** design request/response DTOs the way Surwill does; use validators for normalization; run configuration through `pydantic-settings`; know the v2 sharp edges (a real production incident included).

---

## Why this matters

Pydantic is FastAPI's engine room — every byte in and out passes through it, and
`Settings` runs your twelve-factor config. Interviews probe it three ways:
"where do you validate?", "how do you manage config?", and (senior flavor)
"what changed in Pydantic v2 and what bit you?". Surwill supplies concrete
answers to all three, including a genuine v2 incident.

## Theory

Pydantic models declare fields with type annotations; instantiation **parses**
(coerces) input into those types and raises structured errors otherwise.
Pydantic v2 rebuilt the core in Rust (`pydantic-core`) — parsing is 5–50×
faster than v1, and semantics tightened. Key concepts:

- **Field constraints** — `Field(min_length=8, max_length=128)`; declarative,
  self-documenting, rendered into OpenAPI.
- **Validators** — `@field_validator("email")` for per-field transforms;
  `@model_validator` for cross-field rules.
- **`from_attributes`** (v1's `orm_mode`) — build a model from object
  attributes, enabling ORM→DTO conversion via `model_validate`.
- **DTO discipline** — separate models for input (`ItemCreate`), mutation
  (`ItemUpdate`, all-optional), and output (`ItemResponse`). Never reuse the ORM
  model as the wire format.

> "Pydantic is the most widely used data validation library for Python."
> — Pydantic docs, https://docs.pydantic.dev/latest/

## The code, explained

### Request schemas with normalization (`app/auth/schemas.py`)

```python
class RegisterRequest(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8, max_length=128)
    first_name: str = Field(..., min_length=1, max_length=100)
    last_name: str = Field(..., min_length=1, max_length=100)
    date_of_birth: date | None = Field(default=None)

    @field_validator("email")
    @classmethod
    def normalize_email(cls, v: str) -> str:
        return v.lower()
```

- `EmailStr` (needs the `email-validator` extra — `pydantic[email]` in
  `pyproject.toml`) rejects malformed addresses at the boundary.
- The validator **normalizes rather than rejects**: emails are lowercased once,
  at the edge, so `CASE@x.com` and `case@x.com` are the same account everywhere
  downstream — the DB unique index, login lookups, rate-limit keys. Fixing case
  at every call site instead is the bug factory this design avoids.
- `password` max length matters too: bcrypt truncates at 72 bytes; bounding
  input length keeps hashing costs predictable and honest.

### Update schemas and the tri-state problem

`ItemUpdate`-style models make every field optional. The service then uses:

```python
for field, value in data.model_dump(exclude_unset=True).items():
    setattr(item, field, value)
```

`exclude_unset` distinguishes **"field not sent"** from **"field sent as
null"** — the tri-state that PATCH semantics require. `exclude_none` (used in
`update_item`) is the blunter cousin: it also drops explicit nulls, meaning
"clear this field" can't be expressed. Knowing which one a given endpoint needs
is exactly the kind of detail that separates users of a framework from owners
of one.

### Settings as code (`app/config.py`)

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    APP_ENV: str = "development"
    SECRET_KEY: str                      # no default — must be provided
    DATABASE_URL: str
    REDIS_URL: str = "redis://localhost:6379/0"
    CORS_ORIGINS: list[str] = ["http://localhost:3000"]
    SMTP_HOST: str = ""                  # empty string = feature disabled
    ...

    @property
    def is_production(self) -> bool:
        return self.APP_ENV == "production"

settings = Settings()
```

- Values come from env vars (which win) then `.env`; types are validated at
  import — a missing `SECRET_KEY` kills the process with a clear error at boot,
  not at first request.
- `CORS_ORIGINS: list[str]` is parsed from JSON in the env var
  (`'["http://localhost:3000"]'` in docker-compose) — pydantic-settings handles
  complex types.
- The **empty-string-disables** convention (`SMTP_HOST=""` → emails are logged,
  not sent; `GOOGLE_CLIENT_ID=""` → endpoint disabled) makes features
  cold-startable without config and is friendlier than `None` in env-var land.

Right below the class, config validates *policy*, not just types:

```python
if settings.is_production:
    if settings.SECRET_KEY in _DEV_SECRETS or len(settings.SECRET_KEY) < 32:
        raise RuntimeError("Refusing to start: SECRET_KEY is a development default...")
```

Fail-fast beats a silently-forgeable JWT signing key in production. This
guard-at-boot pattern is worth stealing for every project.

### War story: `extra_forbidden` in production… almost

When the OTP feature was removed, its settings fields were deleted from the
class — but the running containers still had `OTP_EXPIRE_MINUTES=10` etc. in
their environment. The next boot crashed:

```
pydantic_core._pydantic_core.ValidationError: 4 validation errors for Settings
otp_expire_minutes — Extra inputs are not permitted [type=extra_forbidden]
```

Newer `pydantic-settings` defaults to **rejecting unknown env keys** that match
its prefixing rules. The fix was operational, not code: purge the stale env vars
from compose and `.env`. The lesson generalizes: *settings classes and deploy
environments version together* — deleting a field is a breaking change for the
environment. This is a first-class interview story because it shows config as a
contract, and that you read the traceback rather than downgrading the library.

## How this is used in production

Config-as-a-typed-class is the industry default now (12-factor, but validated):
one `Settings` object, environment-injected, no `os.getenv` scattered through
business logic. Teams layer secrets managers (Vault, AWS SM, Railway variables)
on top — the app still just reads env vars. On the DTO side, high-traffic shops
lean on Pydantic v2's Rust core; when even that is hot, they drop
`response_model` on the hottest endpoints and serialize with orjson — a
targeted, measured escape, not a policy.

## Advanced corner

- **v1→v2 rename map** you should recite: `orm_mode`→`from_attributes`,
  `.dict()`→`.model_dump()`, `.parse_obj()`→`.model_validate()`,
  `@validator`→`@field_validator`, class `Config`→`model_config`.
- **Coercion policy:** v2's default ("smart" mode) still coerces `"5"`→`5` for
  `int` fields from JSON strings; `Strict` types / `strict=True` turn that off.
  Know your boundary's coercion story before an auditor asks.
- **Validation is also a DoS surface:** unbounded `str` fields accept megabytes.
  Surwill bounds every human field (`max_length=100/128/255`). Combine with a
  body-size limit at the proxy.
- **Performance:** model *class creation* is expensive; never define models
  inside functions on a hot path. Instantiation is cheap; `model_validate` on
  huge lists is where you profile.

## Best practices

- Separate Create/Update/Response models per resource; never expose ORM models.
- Normalize (lowercase, strip) in field validators at the boundary — once.
- No-default fields for real secrets; boot-time policy guards for prod.
- Bound every string; use `exclude_unset` for PATCH.

## Common mistakes & gotchas

- Reusing the create schema for updates → "all fields suddenly required."
- `exclude_none` when you needed `exclude_unset` → "clear field" impossible.
- Mutable default like `CORS_ORIGINS: list = []` shared across instances —
  Pydantic handles this correctly (deep-copies defaults), unlike dataclasses;
  but don't carry the dataclass scar-tissue reflex into a claim that it's broken.
- Forgetting the `email-validator` extra and shipping `EmailStr` that ImportErrors.

## Where AI helps (and hurts)

AI is superb at generating DTO triads from a table definition. It reliably
produces **v1/v2 chimera code** (`.dict()`, `class Config`) — one grep for
`orm_mode|\.dict\(` after any AI-assisted PR pays for itself.

## Learn independently

- Pydantic docs — "Models", "Validators", and the v2 migration guide:
  https://docs.pydantic.dev/latest/migration/
- pydantic-settings: https://docs.pydantic.dev/latest/concepts/pydantic_settings/
- **Robust Python** (Viafore) — ch. 8–9 on constrained types as documentation.
- **Architecture Patterns with Python** — ch. 3 on the DTO/domain boundary.
- The Twelve-Factor App, factor III (config): https://12factor.net/config

## Interview Q&A

**Q: Where does validation live in your project?**
A: "At the boundary, in Pydantic schemas — types, length bounds, EmailStr, and
normalizing validators like lowercasing emails so the rest of the system never
handles a mixed-case duplicate. Business rules that need the DB (uniqueness,
ownership) live in the service layer and surface as domain exceptions. The
split is: shape at the edge, semantics in services."

**Q: How do you manage configuration and secrets?**
A: "A single pydantic-settings class: typed fields, env-var sourced, `.env` for
dev. Secrets have no defaults so the app can't boot without them, and a
boot-time guard refuses production startup with known dev values or a short
SECRET_KEY. Empty-string means feature-disabled for optional integrations like
SMTP. And I learned the hard way that deleting a settings field is a breaking
change for the environment — newer pydantic-settings rejects unknown keys."

**Q: Pydantic v2 — what actually changed?**
A: "Rust core, so parsing got roughly an order of magnitude faster; API renames
(`model_dump`, `model_validate`, `from_attributes`); stricter semantics and
`extra` handling. Migration is mostly mechanical, but mixed v1/v2 idioms in one
codebase is the common failure mode."

## Exercises against the codebase

- **Easy:** Send `{"email": "  ANA@X.COM "}` variants at `/auth/register`;
  decide whether a `.strip()` belongs in the validator (it does — add it).
- **Medium:** Write `WishItemCreate`'s missing cross-field rule as a
  `@model_validator`: `original_price`, when present, must exceed `price`.
  Add a test.
- **Hard:** Reproduce the war story locally: add a dummy required field to
  `Settings`, boot the container without it, read the error; then remove a field
  while its env var is still set and observe `extra_forbidden`. Write a
  three-line runbook for each failure.

## Key takeaways

- Pydantic = parse at the boundary; normalize once; bound everything.
- Create/Update/Response triad, `exclude_unset` for PATCH tri-state.
- `Settings` is typed config with boot-time policy guards — and it versions with your environment (the `extra_forbidden` war story).
- Know the v2 rename map and its Rust-core performance story cold.
