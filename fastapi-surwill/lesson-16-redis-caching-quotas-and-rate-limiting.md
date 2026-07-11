# Lesson 16 — DEEP DIVE: Redis for Caching, Quotas & Rate Limiting

**Track:** FastAPI (Surwill) · **Lesson:** 16 of 17 · **Estimated time:** 2.5 hours
**Prerequisites:** Lesson 14 (Live Presence — Redis Sorted Sets), basic understanding of TTL/expiry concepts
**Learning objectives:**
- Implement the cache-aside pattern correctly, including cache-key design and cache-busting via key versioning
- Build a rolling-window rate limiter and a fixed-window quota system from Redis primitives (`INCR`/`EXPIRE`), and know which one you're building at any given moment
- Articulate and defend a **fail-open vs. fail-closed** design decision for a specific system, not just recite the terms
- Choose the right Redis data structure (string, sorted set, hash) for a given access pattern
- Understand where a "best-effort cache" stops being safe to treat as best-effort

---

## Why this matters

Lesson 14 covered one Redis data structure — the sorted set — for one purpose — ephemeral presence tracking. That lesson is deliberately narrow, a compressed system-design case study. This lesson is the opposite: it surveys **four different Redis usages across one subsystem** (Surwill's scraper), each solving a genuinely different problem — a metadata cache, a cost-control quota, a distributed rate limiter, and a robots.txt cache — and asks you to notice what's *the same* about all four (the "best-effort, fail-open" design stance) and what's *different* (the data structure and TTL strategy each one needs). Interviewers ask "how would you rate-limit this" and "how would you cache that" as if they're the same question; they're not, and being able to say precisely why is what separates rote Redis usage from actually understanding it.

## Theory

### Redis is not "a database" — it's a toolbox of data structures with O(1)-ish operations

The mental model that unlocks Redis is: it's an in-memory key-value store where the *values* are typed data structures (strings, hashes, lists, sets, sorted sets, streams), and almost every operation on them is O(1) or O(log N). This means the *design* question is never "how do I store this in Redis" — it's "which structure's native operations match the access pattern I need," because picking the right one gets you correctness and performance for free, and picking the wrong one means fighting the structure with application-level workarounds.

### Pattern 1 — Cache-aside with a versioned key prefix

```python
# app/items/scraping/cache.py
_PREFIX = "scrape:cache:v2:"

def _key(url: str) -> str:
    digest = hashlib.sha256(url.encode("utf-8")).hexdigest()
    return f"{_PREFIX}{digest}"

async def get_cached(url: str) -> dict | None:
    try:
        redis = await get_redis()
        raw = await redis.get(_key(url))
        if not raw:
            return None
        return json.loads(raw if isinstance(raw, str) else raw.decode())
    except Exception:
        return None

async def set_cached(url: str, data: dict, ttl_seconds: int) -> None:
    if ttl_seconds <= 0:
        return
    try:
        redis = await get_redis()
        await redis.setex(_key(url), ttl_seconds, json.dumps(data, default=str))
    except Exception:
        return
```

This is **cache-aside** (also called "lazy loading"): the application, not Redis, owns the read-through logic — check the cache, and on a miss, go compute the real answer and populate the cache for next time. Redis itself has no idea a "cache" exists; it just stores a string with a TTL (`SETEX key ttl value` — set-with-expire in one atomic command).

Two design choices worth dwelling on:

**Hashing the URL for the key.** A raw product URL can be arbitrarily long, contain characters that complicate key parsing (`?`, `&`, unicode), and — a subtler point — a plain-text key means anyone with Redis introspection access can enumerate every URL ever scraped just by scanning keys. Hashing with SHA-256 gives a fixed-length, opaque, collision-resistant key. The trade-off: you lose the ability to `SCAN` for "all cached entries for domain X" by key pattern — if you needed that, you'd store the domain in a Redis hash's fields or a side index instead.

**The `v2` in the prefix is a manual cache-generation number, and the comment explains exactly why:**

```python
# v2: bumped when empty results stopped being cached at full TTL — orphans every
# pre-fix entry (including cached failures) without needing a manual Redis flush.
```

This is a real production incident encoded as a one-line fix: an earlier version of this cache was caching *failed/thin* scrapes at the same TTL as successful ones (hours), which meant a transient failure — a site hiccup, a temporary block — got treated as "this URL has no data" for the *entire cache lifetime*, silently preventing every retry from ever reaching the real scraper again. Bumping the prefix from `v1` to `v2` is the cheapest possible way to invalidate every existing cache entry: old keys (`scrape:cache:v1:*`) simply stop being read or written, and Redis's own TTL eventually reclaims them — no `FLUSHDB`, no maintenance window, no risk of also nuking unrelated keys sharing the same Redis instance.

### Pattern 2 — A rolling-window quota with `INCR` + `EXPIRE`

```python
# app/items/scraping/quota.py
_PREFIX = "scrape:managed:quota:"
_WEEK_SECONDS = 7 * 24 * 3600

async def allow_managed_scrape(user_id: uuid.UUID) -> bool:
    limit = settings.MANAGED_SCRAPER_USER_WEEKLY_LIMIT
    if limit <= 0:
        return True
    try:
        redis = await get_redis()
        used = int(await redis.get(_key(user_id)) or 0)
        return used < limit
    except Exception as exc:
        _log.warning("managed-scrape quota check failed (allowing): %s", exc)
        return True

async def consume_managed_scrape(user_id: uuid.UUID) -> None:
    try:
        redis = await get_redis()
        key = _key(user_id)
        used = await redis.incr(key)
        if used == 1:
            await redis.expire(key, _WEEK_SECONDS)
    except Exception as exc:
        _log.warning("managed-scrape quota consume failed: %s", exc)
```

This is a **rolling personal window**, not a calendar week — and the mechanism that gives you that is exactly two commands: `INCR` (atomically increments, creating the key at `1` if it didn't exist) and, *only on the first increment* (`used == 1`), `EXPIRE key 604800`. The key's TTL starts counting down from the user's *first* paid scrape, and the counter resets to zero exactly 7 days after that — not at midnight Sunday, not on a fixed calendar boundary. Each user effectively gets their own independent weekly clock.

The `if used == 1:` check is doing real work: if you called `EXPIRE` on *every* increment, you'd reset the TTL every time, turning this into "no more than N scrapes in any rolling 7-day window measured from the *most recent* scrape" — a materially different (and much stricter) policy than "N scrapes per 7 days starting from the first one." Getting this backwards is a subtle, easy-to-ship bug: both versions "work" in the sense of returning a number under a limit, but they enforce different budgets.

**Why `INCR` and not "read the count, check it, then write count+1" as three separate steps?** Because that's a classic **check-then-act race condition**: two concurrent requests both read `used=9` (limit 10), both decide "9 < 10, allow it," both then write `10` — and you've just let two scrapes through on a budget of one remaining. `INCR` performs the read-modify-write as a single atomic Redis operation; there is no window where two clients can observe the same pre-increment value and both proceed. This is precisely the kind of race a language's `+=` operator does *not* protect you from in a multi-process/multi-worker environment (Surwill's API and worker are separate processes, both potentially calling this) — the atomicity has to come from the datastore, not from the application's control flow.

### Pattern 3 — A fixed-window rate limiter, shared across processes

```python
# app/items/scraping/throttle.py
async def throttle_domain(url: str, rate_per_second: int) -> None:
    host = _host(url)
    if not host or rate_per_second <= 0:
        return
    try:
        redis = await get_redis()
        window = int(time.time())
        key = f"scrape:rate:{host}:{window}"
        count = await redis.incr(key)
        if count == 1:
            await redis.expire(key, 2)
        if count > rate_per_second:
            await asyncio.sleep(1.0)
    except Exception:
        return
```

Same `INCR`+`EXPIRE`-on-first-increment mechanism as the quota, but a completely different *window strategy*: the key itself embeds the current second (`window = int(time.time())`), so a *new* key is implicitly created every second, and the previous second's key just expires on its own (the `EXPIRE key 2` is a safety margin, not the primary reset mechanism — the reset is really "next second, it's a different key"). This is a **fixed window counter** — simpler than a true sliding window (which would need a sorted set of timestamps, like presence in Lesson 14, and a per-request cleanup of entries older than the window), and it has the fixed window's well-known weakness: a burst straddling the boundary between two windows (e.g., `rate_per_second` requests at 0.99s and another `rate_per_second` at 1.01s) can momentarily exceed the intended rate by up to 2x. Surwill accepts this trade-off because the goal here is "don't hammer one domain," not "provide a hard real-time guarantee" — the throttle is a politeness mechanism, not a correctness-critical limiter, so the simpler, cheaper structure is the right choice.

Notice too: this key is **shared across the API process and the worker process** (both import the same `throttle_domain`), which is *why* it has to live in Redis and not, say, a Python `dict` guarded by a lock — an in-process rate limiter would only ever see its own process's traffic, letting the API and worker each independently hammer the same domain at full speed while each individually believes it's being polite.

### The thread that ties all three together: fail-open, deliberately

Every one of these three files has the same shape at the bottom of every function: `try / except Exception: return` (or, for the read side, `return None` / `return True`). This is **fail-open** — if Redis is unreachable, degraded, or slow enough to error, the *caller's primary operation proceeds as if the cache/quota/throttle didn't exist*, rather than blocking or erroring out. The docstrings say this explicitly:

> "Like the scrape cache, this degrades gracefully: if Redis is unreachable the check fails *open* (the scrape proceeds) so a cache outage never breaks the add-item flow."

This is a *deliberate, risk-assessed* choice, not laziness. It's correct here because: (1) these are all secondary, cost/politeness/performance concerns layered on top of a primary flow (scraping a URL) that has real value even without them, and (2) the downside of failing open is bounded and recoverable (a few extra paid scrapes, a few extra requests to one domain) — not catastrophic. Fail-open would be the *wrong* choice for, say, an authorization check ("is this user allowed to see this data") or a payment idempotency key — there, a Redis outage should fail **closed** (deny/block), because the downside of failing open is unbounded (a security hole, a duplicate charge). Knowing which failure mode you want, for *this specific* dependency, in *this specific* call site, is the actual skill — "fail open" and "fail closed" are not a global policy, they're a per-decision judgment call.

## Official documentation quotes

> "SET with an EX or PX option is equivalent to `SET key value EX seconds` / `PX milliseconds`... `SETEX` is an alias for `SET key value EX seconds`, kept for backward compatibility." — Redis documentation, `SETEX`, https://redis.io/docs/latest/commands/setex/

> "INCR is atomic: even multiple clients issuing INCR against the same key will never face a race condition." — Redis documentation, `INCR`, https://redis.io/docs/latest/commands/incr/

> "The generic form of `EXPIRE`... after the timeout has expired, the key will automatically be deleted... Redis keys are expired in two ways: a passive way, and an active way." — Redis documentation, `EXPIRE`, https://redis.io/docs/latest/commands/expire/

> "Redis is often referred to as a data structures server... the type of value determines what operations (called commands) are available." — Redis documentation, *Data Types*, https://redis.io/docs/latest/develop/data-types/

## The code, explained

See the four annotated snippets under **Theory** above — `cache.py`, `quota.py`, and `throttle.py` are each fully walked through there, since the mechanism *is* the theory for this lesson. One more file worth a look:

```python
# app/items/scraping/throttle.py — a second, independent cache-aside example
async def _get_robots_txt(scheme: str, host: str) -> str | None:
    cache_key = f"scrape:robots:{host}"
    redis = None
    try:
        redis = await get_redis()
        cached = await redis.get(cache_key)
        if cached is not None:
            return cached if isinstance(cached, str) else cached.decode()
    except Exception:
        redis = None

    # cache miss (or Redis unavailable) — fetch robots.txt for real
    ...
    if redis is not None:
        try:
            await redis.setex(cache_key, _ROBOTS_CACHE_TTL, text)
        except Exception:
            pass
    return text
```

Notice this one caches the **absence** of a Redis connection differently than `cache.py` does — `redis = None` after a failed `get_redis()` call, checked again before the `setex` at the bottom, so a Redis outage during the *read* doesn't cause a crash trying to *write* to a connection that never opened. It's the same fail-open philosophy, just applied at a slightly different granularity (per-connection-attempt rather than per-function-call) because this function has two separate Redis touchpoints (read, then write) instead of one.

## How this is used in production

- **Cache-aside** is the default caching pattern at nearly every company running a relational database behind a web tier — Twitter/X's early "cache aside with Memcached in front of MySQL" architecture is the canonical public case study; Redis has largely replaced Memcached in new systems because it offers TTLs, richer data structures, and persistence options in one tool. The versioned-key-prefix trick for cache invalidation (`v2:`) is extremely common precisely because it avoids the alternative — a `FLUSHDB`/`FLUSHALL` that also evicts every *unrelated* cache sharing that Redis instance, a classic multi-tenant-cache footgun.
- **Rate limiting** at scale (Cloudflare, API gateways like Kong/Envoy) typically uses more sophisticated algorithms than fixed-window — **token bucket** and **sliding window log/counter** hybrids — specifically to avoid the boundary-burst problem described above, because at internet scale that 2x burst is a real abuse vector, not just a rounding error. Companies exposing public rate-limited APIs (Stripe, GitHub) document their limiter's exact algorithm in their API docs because client authors need to know whether to expect strict enforcement or the softer fixed-window behavior.
- **Quota systems** tied to a paid third-party API (Surwill's Scrapfly budget) mirror how most companies meter *any* metered vendor cost — cloud spend guardrails, LLM token budgets, SMS/email sending caps — the pattern is always "count usage in a fast, cheap store (Redis), gate the expensive call on the count, let the expensive call itself be the source of truth for billing reconciliation" (Redis is the *fast gate*, not the *ledger*).
- **Monitoring**: production Redis-backed caches are typically monitored on hit rate (cache hits / total lookups — a sudden drop signals either a deploy that changed the key format, mass expiry, or genuinely new traffic patterns), memory usage and eviction rate (if `maxmemory-policy` allows eviction, a caching layer that's too small silently starts behaving like a much shorter TTL than configured), and connection pool saturation.
- **Security**: rate limiters and quotas are a first line of defense against both malicious abuse and "friendly fire" (a bug causing a retry storm) — teams commonly layer a *cheap* Redis-backed limiter in front of expensive downstream calls (exactly Surwill's pattern: throttle before scraping, quota before paying) rather than relying solely on the downstream vendor's own rate limiting, because failing fast and locally is both cheaper and gives better error messages than waiting for a 429 from a third party.

## Advanced corner

**Fixed window vs. sliding window vs. token bucket — precisely.** A *fixed window* counter (Surwill's `throttle_domain`) resets hard at a boundary and can burst up to 2x at the seam. A *sliding window log* keeps a timestamp per event (a sorted set, exactly like Lesson 14's presence structure, scored by time) and counts entries within the last N seconds on every check — perfectly accurate, but O(window size) memory per key and requires pruning old entries. A *sliding window counter* approximates the sliding log by weighting the previous fixed window's count proportionally to how much of it overlaps the current lookback period — most of the accuracy of a sliding log, most of the cheapness of a fixed window, and it's what most production rate limiters (including Cloudflare's public documentation of their own approach) actually use. A *token bucket* models a bucket that refills at a steady rate and is decremented per request, naturally allowing bounded bursts while enforcing a long-run average rate — this is the algorithm most API gateways expose to *customers* as "burst limit + sustained rate" because it maps cleanly to an intuitive two-number policy.

**Why `INCR` on a key that doesn't exist yet returns `1`, and why that's exploited here twice.** Redis treats a missing key as an implicit `0` for numeric commands — `INCR` on a nonexistent key creates it and returns `1`. Both `consume_managed_scrape` and `throttle_domain` use `if used/count == 1:` to detect "this is the first increment in this window" *without* a separate `EXISTS` check first — one round trip instead of two, and — critically — no race condition between "check if it exists" and "increment it," because there's only one atomic command involved. This is a small but real pattern worth internalizing: whenever you're tempted to write `if not await redis.exists(key): ... ; await redis.incr(key)`, ask whether `INCR`'s own return value already tells you what you need, atomically, in one round trip.

**Cache stampede (thundering herd) — a problem Surwill's cache-aside as written does *not* solve.** If a popular URL's cache entry expires and 50 concurrent requests all miss at once, all 50 will independently re-scrape the same URL simultaneously — wasted work at best, a burst that trips the throttle or burns quota at worst. Production systems solve this with either a distributed lock (`SET key value NX EX ttl` — only one caller wins the right to repopulate, others wait or serve stale) or "early recomputation" (refresh the cache proactively when it's, say, 90% through its TTL, before it actually expires, so there's never a hard miss under load). Surwill's low request volume for any single product URL makes this a non-issue in practice, but it's exactly the kind of question a senior interviewer asks after you've explained the basic cache-aside pattern: "what happens if this key expires under concurrent load?"

**`SETNX`/`SET ... NX` as a distributed lock, and why it needs more than just "set if not exists."** A naive lock is `SET lock_key 1 NX EX 10` (acquire) and `DEL lock_key` (release) — but a process that acquires the lock, then hangs past the TTL, then wakes up and calls `DEL` unconditionally, can delete a lock that a *different* process has since legitimately acquired. The fix is to set a unique token as the lock's value and only `DEL` if the current value still matches your token (done atomically via a Lua script, since "check value, then delete" is itself a race condition without one). This is the seed of the **Redlock** algorithm for distributed locking across multiple Redis nodes — worth knowing the name and the one-sentence idea even if you never implement it, because "how would you build a distributed lock on Redis" is a recurring senior-level question.

## Best practices

- **Wrap every Redis call in try/except with an explicit, reasoned fail-open or fail-closed decision** — never let "what happens if Redis is down" be an accident of whatever exception propagates.
- **Version your cache key prefixes** (`v1:`, `v2:`) instead of relying on `FLUSHDB` for invalidation — it's surgical, it doesn't affect unrelated keys sharing the instance, and old entries clean themselves up via their own TTL.
- **Use `INCR`'s atomicity instead of read-check-write for any counter that multiple processes/requests can touch concurrently** — this is the difference between a correct rate limiter and one with a race condition that only shows up under real concurrent load.
- **Set the TTL on the *first* increment only** (`if count == 1: expire(...)`) when you want a window that starts from first-use, not one that resets on every access — know which one your feature actually needs.
- **Hash long or sensitive values (like full URLs) into fixed-length cache keys**, both for key-length hygiene and to avoid leaking raw data through key enumeration.
- **Pick the data structure that matches the query you'll actually run** — a string with TTL for "is this cached / expired," a sorted set for "who's active in the last N seconds," a hash for "many fields of one entity read/written independently."

## Common mistakes & gotchas

- **Treating a fixed-window counter as if it were a hard, precise rate limit.** It isn't — it can burst up to ~2x at window boundaries. Fine for politeness throttling, wrong for a promise you're making to a paying customer about their exact rate limit.
- **Resetting a TTL on every `INCR`** when you meant "N per rolling week from first use" — this silently turns into a much stricter "N per rolling week from *most recent* use" policy.
- **Forgetting that a "best-effort, fail-open" cache is not a source of truth** — if code anywhere starts *relying* on the cache being populated (rather than treating a miss as normal and cheap), you've accidentally made a performance optimization into a correctness dependency, and a Redis blip becomes a real outage.
- **Not handling the `bytes` vs `str` decoding** — depending on client configuration, `redis.get()` can return `bytes`, and code that assumes a `str` (`json.loads(raw)` directly, no decode) will crash intermittently depending on how the client was configured. Surwill's `cache.py` handles this explicitly: `raw if isinstance(raw, str) else raw.decode()`.
- **Sharing one Redis instance across unrelated concerns without a key-naming convention**, making it impossible to safely `FLUSHDB` or reason about memory usage per feature. Surwill's `scrape:cache:`, `scrape:managed:quota:`, `scrape:rate:`, `scrape:robots:` prefixes are exactly this discipline in practice.

## Where AI helps (and where it hurts)

**Helps:** scaffolding the boilerplate for a new cache-aside function once you've described the key format and TTL; explaining Redis command semantics (`EXPIRE` vs `PEXPIRE`, `SETEX` vs `SET ... EX`) quickly; suggesting which data structure fits a described access pattern. **Hurts:** deciding fail-open vs. fail-closed for you — that's a risk judgment about *your* specific system's blast radius if the dependency fails, and getting it backwards (failing open on an authorization check, say) is a security bug an AI has no way to know it's introducing unless you tell it the stakes explicitly. Also be skeptical of AI-suggested rate-limiting code that looks like a fixed window but is presented as a precise guarantee — verify which algorithm you're actually getting, because the difference matters and the code often *looks* similar across all of them.

## Learn independently

- **Designing Data-Intensive Applications** (Kleppmann) — Chapter 11 covers the theory of rate limiting and windowing in the context of stream processing; the underlying concepts (fixed vs. sliding windows) transfer directly.
- **Redis official documentation** — *Redis University* free course RU101 ("Introduction to Redis Data Structures") is the best structured primer on choosing the right structure; also read the command reference pages for `INCR`, `EXPIRE`, `SET` (with `NX`/`XX`/`EX` options) directly: https://redis.io/docs/latest/commands/
- **System Design Interview – An Insider's Guide** (Alex Xu, Vol. 1) — Chapter 4 ("Design a Rate Limiter") walks through fixed window, sliding window log, sliding window counter, and token bucket with diagrams — the canonical interview-prep treatment of exactly this lesson's Pattern 3.
- **Cloudflare Engineering Blog** — "How we built rate limiting capable of scaling to millions of domains" (search their blog) — a real production account of the sliding-window-counter approximation mentioned in the Advanced Corner.
- **Redis in Action** (Josiah Carlson) — Chapter 6 covers building locks, counting semaphores, and rate limiters directly on Redis primitives, at a level of detail beyond this lesson.

## Interview preparation

**Q: How would you rate-limit an API endpoint to 100 requests per minute per user?**
A: I'd clarify first whether "100 per minute" needs to be a hard, precise guarantee or a reasonable approximation, because that determines the algorithm. For a simple, cheap approximation, a fixed-window counter — `INCR` a key like `ratelimit:{user_id}:{minute_bucket}`, set a TTL on first increment, reject once the count exceeds 100 — is a few lines and one Redis round trip per check, but it can allow up to ~200 requests in a worst-case two-second window straddling a boundary. For a precise guarantee, I'd use a sliding window counter (weight the previous window's count by overlap) or a token bucket if I want to also allow controlled bursts. I'd pick based on whether this is protecting infrastructure from abuse (fixed window is fine) or is a contractual promise to API customers (needs the more accurate algorithm).

**Q: What does "fail open" mean, and when is it the wrong choice?**
A: Fail open means: if the dependency backing a check (here, Redis) is unavailable, the system proceeds as if the check passed, rather than blocking the operation. It's the right choice when the check is a secondary optimization or cost-control layered on top of a primary flow that has value on its own — Surwill's scrape cache, quota, and throttle are all like this: their failure just means slightly more scraping traffic or paid-tier usage, a bounded and recoverable cost. It's the *wrong* choice for anything where the check IS the safety property — an authorization check, a payment idempotency guard, a distributed lock preventing double-processing — because failing open there means the system silently does the unsafe thing under exactly the conditions (infra trouble) when you'd most want it to be conservative.

**Q: Why is `INCR` safe under concurrency when "read the value, compare it, write value+1" as three separate application-level steps is not?**
A: `INCR` executes as a single atomic operation inside Redis — from the perspective of any two concurrent clients, one `INCR` fully completes before the other begins; there's no interleaving possible. The three-step version has a window between the read and the write where a second client can read the *same* stale value, and both clients then compute and write the same "old value + 1," silently losing an increment (or, in a quota context, letting through one more request than the limit allows). This is the general shape of a check-then-act race condition, and the fix is always the same: push the read-modify-write into a single atomic primitive the datastore provides, rather than orchestrating it from application code across multiple round trips.

**Q: [System design / trade-off] You're asked to add a cache in front of an expensive database query that's called extremely frequently by a popular endpoint. What do you need to decide before writing the cache-aside code?**
A: Several things, roughly in this order: (1) TTL — how stale is acceptable, balancing freshness against cache-hit rate; (2) key design — does it need to be hashed/normalized, and does the key need to encode anything that would make two logically-different queries collide on the same key; (3) invalidation strategy — will I actively invalidate on writes, or purely rely on TTL expiry (simpler, but means writes are visible only after up to one TTL's delay); (4) fail-open vs. fail-closed if the cache backend is unavailable — almost always fail-open (fall through to the real query) for a pure performance cache; (5) stampede protection — if this endpoint is popular enough that many concurrent requests could all miss the cache at once when a key expires, do I need a lock or early-recomputation strategy, or is the traffic low enough that duplicate work on a rare miss is acceptable.

## Exercises against the codebase

**Easy:** `throttle_domain`'s window key is `f"scrape:rate:{host}:{window}"` where `window = int(time.time())`. Trace through what happens, second by second, for a burst of 15 requests to the same domain arriving in the first 100ms of a fresh second, with `rate_per_second = 5`. How many get delayed, and by how much (approximately)?

**Medium:** `allow_managed_scrape` and `consume_managed_scrape` are two separate calls — meaning there's a window between "check the quota" and "consume it" where a different concurrent request for the *same user* could also pass the check before either consumes. Is this a real race condition given how these functions are actually called in `app/items/services.py` (look at the call sites)? If it is, sketch a fix using a single atomic Redis operation that both checks and increments in one round trip.

**Hard:** Design (in prose + pseudocode, not necessarily runnable) a cache-stampede guard for `get_cached`/`set_cached` in `cache.py`, using `SET key value NX EX ttl` as a distributed lock so that when a popular URL's cache entry expires under concurrent load, only *one* caller re-scrapes while the others either wait briefly or fall through to scraping anyway after a short timeout (your choice — justify it). Explain what happens if the lock-holder crashes before releasing the lock, and why the lock's own TTL matters for that failure mode.

## Key takeaways

- Redis's power is its typed data structures with atomic, near-O(1) operations — the design question is always "which structure matches this access pattern," not "how do I shoehorn this into a key-value store."
- Cache-aside, quota, and rate-limiting are three *different* problems that happen to share the same two primitives (`SET`/`SETEX` and `INCR`/`EXPIRE`) — know which one you're building and why the window strategy (rolling-from-first-use vs. fixed-window-per-second) differs between them.
- `INCR`'s atomicity is what makes it safe under concurrency — a hand-rolled read-then-write is a race condition waiting for enough load to expose it.
- Fail-open vs. fail-closed is a per-decision judgment call based on blast radius, not a global policy — get this backwards on a security-relevant check and "graceful degradation" becomes a vulnerability.
- Versioned cache-key prefixes are a surgical, zero-downtime invalidation tool that beats `FLUSHDB` for any Redis instance shared across features.
- A fixed-window rate limiter is simple and usually good enough for internal politeness throttling; know the sliding-window and token-bucket alternatives by name and trade-off for when "good enough" isn't.
