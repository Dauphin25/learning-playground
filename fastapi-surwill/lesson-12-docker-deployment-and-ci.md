# Lesson 12 — Docker, Deployment & CI

**Track:** FastAPI (Surwill) · **Time:** ~2h · **Prerequisites:** lessons 01–02, 07, 11
**Objectives:** read the Dockerfiles and compose files critically, understand the five-container topology, the Railway deploy plan, the CI workflows, and the production trade-offs the codebase documents.

---

## Why this matters

"How is it deployed?" closes most system-design interviews. A good answer covers
the container topology, how config/secrets flow, how migrations run, how CI
gates merges, and — the senior part — which corners you *knowingly* cut and why.
Surwill ships with real Dockerfiles, a full-stack compose, GitHub Actions, and a
written `DEPLOY.md`, so you can speak from artifacts, not aspirations.

## Theory

A container bundles the app with its runtime and OS deps into an image that runs
identically on your laptop and in prod — the reproducibility that kills "works
on my machine." **Compose** orchestrates multiple containers as one stack.
**CI** runs tests/build on every push so main stays releasable. **CD** promotes
green builds to an environment (here, Railway from GitHub).

Two principles thread the whole lesson: *build once, run anywhere* (config comes
from the environment, not the image) and *fail before deploy, not after* (CI is
the gate).

## The code, explained

### Backend Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install uv
COPY pyproject.toml .
COPY app ./app
RUN uv pip install --system -e .
RUN python -m playwright install chromium --with-deps
COPY . .
CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2 --proxy-headers --forwarded-allow-ips '*'"]
```

- **`python:3.12-slim`** — small Debian base; not `alpine` (musl breaks many
  wheels, notably scientific/parsing libs).
- **`uv`** — a Rust-based, much faster pip replacement; the ecosystem's current
  direction.
- **Dependency-copy before code** *would* cache the install layer across code
  changes — note this Dockerfile copies `app` before the install for the
  editable install, a minor caching compromise worth mentioning as improvable.
- **Playwright Chromium `--with-deps`** — pulls a headless browser **and** its
  OS libraries in one cached layer. This is the single biggest thing in the
  image (hundreds of MB) and only the scraper uses it — the documented
  first-optimization (split into a scraper worker image).
- **Pinned dependencies** (`==` in `pyproject.toml`, lesson 04/07) — a rebuild
  can't silently pull a breaking major.
- **CMD** migrates then serves with 2 workers and proxy-header trust (lessons
  02, 06, 07 all converge here).

### Frontend Dockerfile — multi-stage

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
USER app
CMD ["node", "server.js"]
```

The **multi-stage** pattern: the `builder` stage has the full toolchain and
`node_modules`; the final stage copies only Next.js **standalone** output (a
minimal server + just the needed deps). Result: a much smaller, faster-starting
production image that **runs as a non-root user** (`USER app`) — defense in
depth if the process is compromised. `npm ci` (not `npm install`) installs
exactly the lockfile for reproducible builds.

### The full-stack compose — five containers

`E:\surwill\docker-compose.yml` wires `db`, `redis`, `api`, `worker`,
`frontend`. The instructive parts:

```yaml
depends_on:
  db:    { condition: service_healthy }
  redis: { condition: service_healthy }
```

`condition: service_healthy` means the API waits for Postgres/Redis
**healthchecks** to pass, not merely to start — no more "DB not ready" boot
crashes. The `api` and `worker` build from the **same context** with different
commands (lesson 01's one-image/two-entrypoints). Secrets come from the
environment (`SECRET_KEY`, `GOOGLE_CLIENT_ID`), and a subtle detail:

```yaml
frontend:
  build:
    target: builder    # dev uses the toolchain stage for `npm run dev`
```

The compose targets the *builder* stage so the dev server has the full
toolchain, while production builds the final slim stage — one Dockerfile, both
needs.

### CI (`.github/workflows/ci.yml`)

Backend:

```yaml
services:
  postgres: { image: postgres:16-alpine, env: {...}, ports: ["5432:5432"], options: --health-cmd ... }
  redis:    { image: redis:7-alpine, ports: ["6379:6379"], options: --health-cmd ... }
env:
  TEST_DATABASE_URL: postgresql+asyncpg://surwill:surwill@localhost:5432/surwill_test
  SECRET_KEY: ci-secret-key-not-used-in-production-1234
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-python@v5
    with: { python-version: "3.12" }
  - run: pip install uv && uv pip install --system -e ".[dev]"
  - run: ruff check app        # continue-on-error: legacy, tighten later
  - run: pytest -q
```

The **service containers** are why lesson 11's env-overridable
`TEST_DATABASE_URL` exists — CI points the same conftest at its own Postgres.
Frontend CI runs `npm ci` + `npm run build` (which type-checks, now that
`ignoreBuildErrors` was removed — lesson mirrors the frontend hardening). `ruff`
is `continue-on-error` for now: an honest "we lint but haven't zeroed legacy
violations yet," better than pretending.

### The deploy plan (`DEPLOY.md`, Railway)

The repo ships a written checklist: managed Postgres+Redis (backups included),
`api` + `worker` + `frontend` services from the two repos, every env var
enumerated, custom domains with automatic HTTPS, Google OAuth origins, SMTP
setup, and a first-deploy smoke test. Two decisions worth internalizing:

- **`NEXT_PUBLIC_API_URL` is a build-time var** — Next.js inlines
  `NEXT_PUBLIC_*` at build, so it must be set *before* the frontend build, not
  at runtime. A classic gotcha the checklist calls out.
- **Migrate-on-boot** is fine for Railway's single-replica default; the
  multi-replica release-phase move is documented as the scale-up step (lesson 07).

## How this is used in production

Startups deploy almost exactly this: managed Postgres/Redis + container services
on Railway/Render/Fly, GitHub Actions gating merges, HTTPS terminated by the
platform. Enterprises swap the platform for Kubernetes (Helm/Kustomize),
migrations into an init-container or Argo hook, images scanned (Trivy) and signed
(cosign), and add staging→canary→prod promotion. The *concepts* are identical —
Surwill's compose `depends_on: service_healthy` is a readiness gate,
conceptually the same as a Kubernetes readiness probe; its `/api/health`
(lesson 02) is what the probe would hit.

## Advanced corner

- **Image size is attack surface + cost + cold-start.** The Playwright layer
  makes the API image large; splitting the scraper into its own worker image
  slims the API, shrinks its attack surface (no browser), and speeds API
  cold-starts. This is *the* documented next step — a great "what would you
  improve" answer.
- **Layer caching economics.** Order Dockerfile steps stable→volatile: base,
  system deps, dependency manifests+install, *then* app code. Surwill's backend
  copies `app` before the editable install, so app edits bust the dep layer —
  a real, nameable improvement.
- **Non-root everywhere.** The frontend runs as `app`; the backend still runs as
  root (base default) — a hardening gap worth naming and a one-line fix
  (`USER`), left because Playwright's `--no-sandbox` complicates it.
- **12-factor config discipline.** Same image across environments; only env vars
  differ. The boot-time secret guard (lesson 04) enforces that prod can't run
  with dev secrets — config validated, not just documented.
- **Supply chain.** Pinned Python deps (good), but no lockfile hash pinning or
  image digest pinning yet; SBOM/scanning would be the enterprise addition.

## Best practices

- Multi-stage builds; slim/standalone runtime; non-root user; `npm ci`/pinned deps.
- Healthcheck-gated `depends_on`; same image for API and worker.
- Config/secrets from the environment; validated at boot; never baked into the image.
- CI with real service containers gating every merge; build (with type-check) as a test.
- Write the deploy runbook down (`DEPLOY.md`) — launch by checklist, not memory.

## Common mistakes & gotchas

- `alpine` for Python → musl wheel pain; use slim.
- Setting `NEXT_PUBLIC_*` at runtime and wondering why the frontend still calls localhost — it's inlined at build.
- `depends_on` without `condition: service_healthy` → boot races on cold start.
- Baking secrets into the image (they persist in layers, forever).
- Multi-replica migrate-on-boot → racing DDL (lesson 07).
- Shipping the toolchain in the runtime image (no multi-stage) → bloated, more CVEs.

## Where AI helps (and hurts)

AI writes Dockerfiles, compose, and Actions YAML well and knows the multi-stage
and healthcheck idioms. It won't reason about *your* image's cost drivers (the
Playwright layer), your migration-on-boot replica risk, or your secret-handling
posture — the trade-offs are yours. It also loves to leave the runtime as root;
check `USER`.

## Learn independently

- Docker docs — multi-stage builds; Dockerfile best practices:
  https://docs.docker.com/build/building/multi-stage/
- GitHub Actions — service containers:
  https://docs.github.com/actions/using-containerized-services/about-service-containers
- Next.js "Deployment" + standalone output docs.
- Railway docs (build/deploy, variables): https://docs.railway.app/
- **The DevOps Handbook** (Kim et al.) for the CI/CD philosophy; **Site
  Reliability Engineering** (Google, free) for health checks & release
  engineering; The Twelve-Factor App for config/build/run separation.

## Interview Q&A

**Q: Describe how your app is deployed.**
A: "Five containers — Postgres, Redis, the FastAPI API, an arq worker, and a
Next.js frontend. API and worker build from the same image with different
commands, so their code and models can't drift. Config and secrets come from the
environment and are validated at boot — the app refuses to start in production
with dev secrets. Migrations run before the server via the container command,
which is correct for a single replica; scaling out moves that to a release
phase. Target platform is Railway with managed Postgres/Redis for backups, from
two GitHub repos, with GitHub Actions running the test suite against Postgres and
Redis service containers on every push."

**Q: Your API image is large. Why, and what would you do?**
A: "It bundles Playwright's Chromium plus OS libraries for the scraper's headless
fallback — hundreds of megabytes only the scraping path uses. I'd split scraping
into its own worker image so the API image drops the browser entirely: smaller,
faster cold starts, and a reduced attack surface. It's the first infra
optimization I've documented."

**Q: What does CI check, and where does it fall short today?**
A: "Backend: install with uv, ruff lint, and pytest against Postgres/Redis
service containers. Frontend: `npm ci` and a production build, which
type-checks now that we stopped ignoring build errors. The honest gap is ruff
runs `continue-on-error` because legacy violations aren't zeroed yet, and
there's no image scanning or coverage gate — next steps, not pretend-passes."

## Exercises against the codebase

- **Easy:** `docker images` before/after adding a `RUN du -sh /ms-playwright`
  step; quantify the browser layer's size.
- **Medium:** Reorder the backend Dockerfile so dependency install is cached
  independently of app-code changes; prove the cache hit by editing a route and
  rebuilding.
- **Hard:** Write a second Dockerfile (or build target) that produces a
  browser-less API image and a separate scraper-worker image with Chromium;
  update compose to run three backend services. Measure the API image size
  reduction.

## Key takeaways

- Five containers; API/worker one image, two commands; healthcheck-gated startup.
- Multi-stage + standalone + non-root for the frontend; slim base + pinned deps + Playwright layer for the backend.
- Config from env, validated at boot; `NEXT_PUBLIC_*` is build-time.
- CI gates merges with real service containers (the reason `TEST_DATABASE_URL` is env-overridable).
- Know the documented cuts: Playwright image split, backend non-root, multi-replica migrations — naming them is the senior move.
