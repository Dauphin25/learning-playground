# Lesson 09 — DEEP DIVE: Background Jobs with arq

**Track:** FastAPI (Surwill) · **Time:** ~3h · **Prerequisites:** lessons 02, 05, 06
**Objectives:** know *why* work leaves the request path, how arq's Redis queue works under the hood, every job and cron in `app/worker.py`, the delivery-semantics story (at-least-once, idempotency), and the failure modes you'll be asked about.

---

## Why this matters

"How do you handle long-running work?" is a guaranteed interview question, and
"we used Celery/arq" is only the first sentence of a good answer. The follow-ups
— *what happens if the worker dies mid-job? can a job run twice? how do crons
avoid overlapping? why a queue instead of `BackgroundTasks`?* — separate people
who configured a library from people who understand distributed work. Surwill's
worker is small enough to know completely and real enough to defend.

## Theory

### Why work leaves the request path

A request handler should do only work whose latency the user must pay. Three
categories must leave:

1. **Slow third-party I/O** — sending an email (SMTP handshake: hundreds of ms),
   scraping a store page (seconds, sometimes a headless browser).
2. **Scheduled work** — hourly price refresh, nightly hard-delete sweep. No
   request exists to attach it to.
3. **Retryable work** — an email that fails should retry with backoff; a request
   handler can't hang around to do that.

The generic architecture is **producer → broker → consumer**: the API enqueues
a small message ("send reset email to X with link Y") into Redis and returns
immediately; a separate worker process pops and executes. The queue decouples
*availability* (API up even if worker down — jobs wait), *latency* (user never
waits on SMTP), and *scale* (add consumers without touching the API).

### Why arq (vs Celery, vs FastAPI BackgroundTasks)

- **`BackgroundTasks`** (FastAPI built-in) runs a callable *in the same process
  after the response*. No persistence: process dies, task lost; no retry, no
  schedule, and heavy tasks compete with request handling. Fine for "fire a
  webhook"; wrong for anything that must happen.
- **Celery** — the incumbent: huge feature surface (routing, chords, multiple
  brokers), but sync-first (async support is bolted on), heavy config.
- **arq** — built by the pydantic author *for asyncio*: jobs are `async def`
  functions, Redis is the only broker, retries and cron built in, the whole
  library is readable in an afternoon. For an async FastAPI monolith with one
  Redis already in the stack, arq is the right-sized tool — that sentence is
  your interview justification.

> "Job queues and RPC in python with asyncio and redis."
> — arq docs, https://arq-docs.helpmanual.io/

### How arq actually works (under the hood)

Understanding this is what makes you credible:

- **Enqueue**: `enqueue_job("send_password_reset_email", email, link)` pickles
  the function name + args into a Redis hash (`arq:job:<id>`) and pushes the id
  onto a sorted set scored by execution time (`arq:queue`). Jobs get a random
  id (or one you set — see idempotency below).
- **Poll**: the worker polls the sorted set for due jobs, moves each into an
  in-progress state, and runs the coroutine with a semaphore capping
  concurrency (`max_jobs`, default 10) — one process, many interleaved jobs,
  same event-loop economics as the API.
- **Ack/retry**: success records a result key; failure re-enqueues with a retry
  counter until `max_tries` (default 5); a worker that dies mid-job leaves the
  job to be picked up again after its timeout — i.e. **at-least-once delivery**.
- **Cron**: the worker computes next-run times for `cron()` entries and
  enqueues them itself; a distributed lock (Redis SETNX-style) keeps multiple
  workers from double-firing the same cron tick.

## The code, explained: `app/worker.py`

### The import block that isn't decoration

```python
# Import every model module up front so SQLAlchemy can resolve string-named
# relationships (e.g. Board→"User") no matter which job's imports run first.
from app.access import models as _access_models          # noqa: F401
from app.auth import models as _auth_models              # noqa: F401
from app.boards import models as _boards_models          # noqa: F401
...
```

The worker doesn't go through `app/main.py`, so nothing imports the model
modules for it. SQLAlchemy resolves `relationship("User")` strings lazily
against a registry of *imported* classes — a job that only imported `Item`
would blow up with `InvalidRequestError` the first time a mapper touched
`Board.owner`. This is the vertically-sliced monolith's one global: the ORM
registry. (Same reason `alembic/env.py` imports everything — lesson 07.)

### Job 1: transactional email

```python
async def send_password_reset_email(ctx: dict, email: str, reset_link: str) -> None:
    """Deliver the password-reset link (SMTP; logged when unconfigured)."""
    from app.shared.email import send_password_reset_email as _send
    await _send(email, reset_link)
```

And the producer side (`app/auth/routes.py`):

```python
if raw_token:
    reset_link = f"{settings.FRONTEND_URL}/reset-password?token={raw_token}"
    try:
        arq = await get_arq()
        await arq.enqueue_job("send_password_reset_email", email, reset_link)
    except Exception:
        logger.exception("Failed to enqueue password-reset email")
```

Three deliberate choices:

1. **String job name, primitive args.** The queue carries `("send_password_reset_email",
   "a@b.com", "https://…")` — not objects, not ORM entities. Messages must
   survive serialization and mean the same thing in a different process later.
2. **Enqueue failures don't break the request.** If Redis is down, the
   exception is logged and the user still gets the (enumeration-proof)
   response. Trade-off owned: availability of the endpoint over guaranteed
   email delivery. The alternative (fail the request) is defensible too — the
   point is choosing consciously.
3. **The email function is a no-op-with-log when SMTP is unconfigured**, so the
   worker behaves identically in dev, just without side effects.

The actual sending (`app/shared/email.py`) has its own lesson-worthy line:

```python
await asyncio.to_thread(_send_sync, msg)   # smtplib is blocking — keep it off the event loop
```

The worker is an event loop too! Ten concurrent jobs on one loop means one
blocking `smtplib` call freezes the other nine. `asyncio.to_thread` pushes the
blocking work to a threadpool thread. Any blocking library inside an async
worker gets this treatment.

### Job 2: scrape an item's URL after creation

```python
async def scrape_item_url(ctx: dict, item_id: str, url: str) -> None:
    metadata = await scrape_url_metadata(url)
    async with AsyncSessionLocal() as db:
        result = await db.execute(select(Item).where(Item.id == uuid.UUID(item_id)))
        item = result.scalar_one_or_none()
        if item:
            if metadata.title and not item.title:
                item.title = metadata.title
            if metadata.image_url and not item.image_url:
                item.image_url = metadata.image_url
            await db.commit()
```

Study the defensive shape — it encodes at-least-once thinking:

- **`item_id` is a string**, re-parsed to UUID: queue messages are plain data.
- **The worker opens its own session** (`AsyncSessionLocal()` directly, not the
  request-scoped dependency — there is no request).
- **`if item:`** — the item may have been deleted between enqueue and
  execution. Jobs run *later*, against *changed state*; every job must re-check
  its premises.
- **`and not item.title`** — only fill gaps, never overwrite. This makes the
  job **idempotent**: running it twice (at-least-once delivery!) produces the
  same result as once. Idempotency is the single most important word in this
  lesson.

### Jobs 3–4: the refresh pipeline (fan-out with a semaphore)

`refresh_stale_items_job` (hourly cron + fallback) and `refresh_user_items_job`
(enqueued when someone opens a profile) both call into
`app/items/services.py`, where the real pattern lives:

```python
sem = asyncio.Semaphore(5)

async def _refresh_one(item: Item) -> None:
    async with sem:
        item.status_checked_at = now          # stamp FIRST — even failures count as checked
        try:
            result = await check_item_availability(item.product_url)
        except Exception:
            return
        ...update stock/price, collect notification events...

await asyncio.gather(*[_refresh_one(i) for i in checkable])
await _dispatch_refresh_notifications(events, db)
await db.commit()
```

Five production techniques in fifteen lines:

1. **Bounded concurrency.** `gather` alone would fire 100 scrapes at once;
   the semaphore caps at 5 in flight — polite to stores, gentle on memory.
2. **Stamp-before-work.** `status_checked_at = now` is set even if the scrape
   fails, so a permanently-broken URL doesn't get retried every single pass —
   it waits its full TTL like everything else. Cheap poison-pill defense.
3. **Batched selection.** The cron variant selects `LIMIT 100`, never-checked
   first, then longest-stale (`ORDER BY status_checked_at IS NULL DESC, status_checked_at ASC`)
   — a work-sharing scheme that guarantees progress without unbounded batches.
4. **Concurrent I/O, single-writer DB.** The scrapes run concurrently, but all
   mutations happen on one session, committed once. (An `AsyncSession` is not
   task-safe; this split is the correct shape — gather the *network* work,
   serialize the *state* work.)
5. **Noise-gated notifications.** Price-drop events fire only on drops ≥5%
   (`PRICE_DROP_NOTIFY_MIN_PCT`), stock events only on real transitions with a
   known baseline (`old is not None and new != old`), and first-ever checks
   never notify. Every threshold exists because hourly scraping is *noisy* —
   currency rounding, per-variant prices. Notification spam is a product bug.

### The schedule

```python
class WorkerSettings:
    functions = [send_password_reset_email, scrape_item_url,
                 refresh_stale_items_job, refresh_user_items_job]
    cron_jobs = [
        cron(hard_delete_expired,    hour={3}, minute={0}),   # nightly GDPR-ish purge
        cron(refresh_stale_items_job, minute={0}),            # hourly, top of the hour
        cron(archive_bought_items,   hour={4}, minute={0}),   # bought → archive after 7 days
    ]
    redis_settings = RedisSettings.from_dsn(settings.REDIS_URL)
```

`WorkerSettings` is arq's entrypoint contract — `python -m arq
app.worker.WorkerSettings` reads it. `functions` are the enqueue-able jobs (by
name!); `cron_jobs` are self-scheduling. Note `hard_delete_expired`: soft
deletes (lesson 06) are only safe *because* this reaper exists — soft-delete
without a purge policy is a data-retention liability, a nice GDPR-adjacent
point to raise yourself.

### Producer plumbing

```python
# app/deps.py
_arq_pool: ArqRedis | None = None

async def get_arq() -> ArqRedis:
    global _arq_pool
    if _arq_pool is None:
        _arq_pool = await create_pool(RedisSettings.from_dsn(settings.REDIS_URL))
    return _arq_pool
```

Same lazy-singleton pattern as the Redis client: the enqueue pool is
process-wide, built on first use, hidden behind a callable so tests can
override it.

## How this is used in production

This produce/consume shape is universal; only the broker changes with scale:
Redis queues (arq/Celery/RQ/Sidekiq/BullMQ) → RabbitMQ when routing and
per-message acks matter → Kafka/SQS when volume and replay matter. The
operational kit around it is always the same and worth reciting: **queue-depth
and job-latency metrics** (alert when depth grows monotonically), **dead-letter
handling** for jobs that exhaust retries, **idempotency keys** end to end, and
**graceful drain on deploy** (arq's SIGTERM handling: stop popping, finish
in-flight — visible in Surwill's own logs: `shutdown on SIGTERM ◆ 5 jobs
complete ◆ 1 failed`). Companies at Netflix/Uber scale wrap the same concepts
in internal platforms (e.g. cron-as-a-service with distributed locks), but the
interview vocabulary is identical.

## Advanced corner

- **Delivery semantics.** Exactly-once delivery does not exist across process
  boundaries (Two Generals); systems give you at-least-once (arq: job survives
  worker death, may re-run) or at-most-once. The engineering answer is
  at-least-once **+ idempotent handlers** — which is why every Surwill job
  re-checks state and gap-fills. Say "effectively-once via idempotency", not
  "exactly-once".
- **Custom job IDs as dedup keys.** arq lets you set `_job_id`; enqueueing the
  same id twice while the first is pending is a no-op. `scrape_item_url` could
  use `_job_id=f"scrape:{item_id}"` to collapse duplicate enqueues from
  double-clicks. Good "what would you improve" material.
- **Backpressure.** If producers outrun consumers, Redis grows without bound.
  Mitigations: cap queue depth and shed load, scale consumers, or move the
  work's trigger (Surwill's profile-open refresh is naturally rate-limited by
  the stale-TTL check — items checked within the hour are excluded at SELECT
  time).
- **Priorities & queues.** One queue means the hourly refresh can delay a
  password-reset email by minutes. arq supports multiple queues
  (`queue_name`); a `critical` queue for user-facing email with its own worker
  is the first scaling move. Volunteer it.
- **Poison pills.** A job that crashes deterministically will retry
  `max_tries` times and land in failed results. The stamp-before-work trick
  and try/except-per-item in the refresh pipeline are local antidotes;
  globally you want failed-job alerting.

## Best practices

- Queue messages = names + primitive args; never objects, never entities.
- Every handler idempotent; every handler re-validates state ("does this item still exist?").
- Bounded concurrency for fan-out; single session/single commit for state.
- Blocking libraries via `asyncio.to_thread`; cron for schedules, jobs for events.
- Same image as the API — worker code can never drift from model code.

## Common mistakes & gotchas

- Using `BackgroundTasks` for must-happen work (email!) — process restart = silent loss.
- Sharing one `AsyncSession` across gathered tasks — corrupt state, cryptic errors (lesson 11's war story is the test-suite variant).
- Non-idempotent handlers under at-least-once delivery — double emails, double charges.
- No queue monitoring: the first symptom of a dead worker is a user asking why emails stopped — instrument depth *before* that.
- Forgetting the model-import block in a new worker module and meeting `InvalidRequestError` in production at 3 a.m.

## Where AI helps (and hurts)

AI scaffolds job functions and WorkerSettings instantly. It will *not* make
your handlers idempotent, notice missing state re-checks, or reason about
delivery semantics — those require adversarial thinking about time and failure,
which is exactly what interviews test. Use AI for the skeleton, then interrogate
every handler: "what happens if this runs twice? five minutes late? after the
row was deleted?"

## Learn independently

- arq docs (short, read fully): https://arq-docs.helpmanual.io/
- **Designing Data-Intensive Applications** (Kleppmann) — ch. 8 (faults, time)
  and the "exactly-once" discussion; the intellectual backbone of this lesson.
- **Architecture Patterns with Python** — ch. 8–11 (events & message bus) for
  the producer/consumer shape inside clean architecture.
- Sidekiq's "Best Practices" wiki (Ruby, but the queue wisdom is universal):
  idempotency, small payloads, transactional enqueue.

## Interview Q&A

**Q: Walk me through what happens after a user clicks "forgot password".**
A: "The API generates a CSPRNG token, stores its hash with a 60-minute expiry,
and enqueues `send_password_reset_email` with just the address and link — the
response returns immediately and identically whether the email exists, to block
enumeration. The arq worker pops the job, and sends over SMTP via
`asyncio.to_thread` because smtplib blocks. If Redis is briefly down at enqueue
time we log and still return 200 — endpoint availability over delivery
guarantee, consciously. If the worker is down, jobs queue until it returns."

**Q: Your worker died halfway through a job. What happens?**
A: "arq is at-least-once: the job wasn't acked, so it re-runs when a worker
returns. That's why every handler is idempotent — the scrape job only fills
missing fields and re-checks the item still exists; running it twice converges
to the same state. Exactly-once across processes isn't a real thing, so the
design goal is effectively-once through idempotent handlers."

**Q: How does the hourly refresh avoid hammering stores or the DB?**
A: "Three brakes: SELECT takes a bounded batch (100, never-checked first, then
longest-stale); a semaphore caps concurrent scrapes at 5, with a per-domain
Redis rate limiter below that; and items are stamped checked *before* the
scrape so failures wait a full TTL instead of retrying every pass. Scrapes run
concurrently but all DB writes happen on one session with one commit, because
AsyncSession isn't task-safe."

**Q: Why arq over Celery?**
A: "Async-native — jobs are coroutines on an event loop, matching the rest of
the codebase; Redis-only, which we already ran; retries and cron built in; and
small enough to actually read. Celery wins when you need its routing/workflow
machinery or a Rabbit/SQS broker — we didn't."

## Exercises against the codebase

- **Easy:** `docker compose logs worker -f`, trigger a password reset, and read
  the job lifecycle lines (`→ enqueued/started`, `← finished ●`). Then stop the
  worker, trigger another reset, restart, and watch the queued job drain.
- **Medium:** Add a `send_welcome_email` job fired on registration. Make it
  idempotent (job id keyed on user id) and prove double-enqueue collapses.
- **Hard:** Simulate the poison pill: make `scrape_item_url` raise for one URL,
  observe retries and the failure record; then add `max_tries` + a log-based
  "dead letter" alert. Bonus: implement the `critical` email queue with a
  second WorkerSettings and compose service.

## Key takeaways

- Queue = decoupled availability, latency, scale; arq = async-native, Redis-backed, right-sized.
- At-least-once delivery is the contract; **idempotent handlers** are the response — gap-fill, re-check, stamp-before-work.
- Fan-out with a semaphore; single-session single-commit for state; `to_thread` for blocking libs.
- Cron for schedules with distributed-lock dedup; soft delete demands a reaper.
- Own the failure modes out loud: dead worker, double delivery, poison pill, queue growth.
