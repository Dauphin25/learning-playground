# Lesson 14 — DEEP DIVE: Live Presence ("N people on this page right now") with Redis Sorted Sets

**Track:** FastAPI (Surwill) · **Time:** ~2.5h · **Prerequisites:** lessons 03 (routing), 05 (dependency injection), 08 (auth)
**Objectives:** understand presence as an *ephemeral-state* problem, why it lives in Redis and not Postgres, how one sorted set + a heartbeat gives you a self-cleaning "who's online" system, every line of Surwill's implementation (frontend loop → optional-auth route → sorted-set service → avatar strip), and the scaling/consistency story an interviewer will push you on.

---

## Why this matters

"Show how many users are online right now" looks like a toy feature and is
actually a compressed system-design interview: it forces you to talk about
**ephemeral vs durable state**, **heartbeats and TTLs**, **polling vs push**,
**clock trust**, and **horizontal scale** — all in one endpoint. It's also a
pattern you will re-implement forever: "active drivers on the map", "agents
online in the call center", "collaborators in this document", "players in
lobby" are all the same problem wearing different shirts.

Surwill's landing page shows a strip of real profile photos with a green
`● ON THIS PAGE RIGHT NOW` label and a `+N` overflow tile. The entire feature
is: **one POST endpoint, one Redis sorted set, one `setInterval` on the
client**. No WebSockets, no message broker, no new table. Knowing *why* that
minimal shape is correct — and exactly where it stops being correct — is the
lesson.

## Theory

### Presence is ephemeral state — pick the store accordingly

The defining property of presence data: **it is only true for a few seconds,
and losing it costs nothing** — the next heartbeat rebuilds it. That profile
puts it at the extreme opposite end of the spectrum from your `users` table:

| Property | Durable state (Postgres) | Presence (Redis) |
|---|---|---|
| Lifetime | years | ~2 minutes |
| Loss tolerance | none (backups, WAL) | total (self-heals in one poll cycle) |
| Write rate | low, transactional | 1 write per user per 45 s, fire-and-forget |
| Read shape | rich queries, joins | "give me the freshest N members" |
| Consistency need | strong | bounded staleness is fine |

Writing a heartbeat per user per minute into Postgres would generate dead
tuples at a constant rate (every heartbeat is an UPDATE → old row version →
vacuum churn), pollute the WAL and backups with data that is worthless three
minutes later, and still need a scheduled job to expire stale rows. Redis
holds it in memory, expires it as a side effect of reading it, and forgets it
on restart — which for presence is a *feature*, not a risk.

### The heartbeat + TTL model

A client cannot reliably tell you "I'm leaving" — tabs are closed, laptops
sleep, trains enter tunnels. Every real presence system therefore inverts the
logic: **clients repeatedly assert "I'm here"; the server treats silence as
absence.** Two parameters define the system:

- **Heartbeat interval** `H` — how often the client re-asserts (Surwill: 45 s).
- **TTL** `T` — how long the server believes an assertion (Surwill: 120 s).

The invariant you must maintain: **`T` comfortably > `H`** — Surwill's ratio
of ~2.7× tolerates one dropped heartbeat plus network jitter and browser
timer throttling before a user flickers offline. The trade-off is *ghost
time*: after a user actually leaves, they remain "present" for up to `T`
seconds. Presence is therefore **eventually consistent with bounded
staleness**: the count you display is correct as of at most `H + T` seconds
ago, and everyone accepts this — nobody audits an online counter.

### Why a sorted set (and not the "obvious" alternatives)

Redis gives you several ways to model "set of users with freshness":

1. **One key per user** (`presence:<uid>` with `EXPIRE 120`). Expiry is free,
   but *listing* who is online now requires `SCAN`/`KEYS` over the keyspace or
   a secondary index set — which then needs its own cleanup, recreating the
   problem. Wrong shape for "read the whole roster on every request".
2. **A plain SET of user ids.** Members of a set can't carry timestamps, and
   Redis has **no per-member TTL** inside one key — you could never expire
   individuals.
3. **A HASH of `uid → timestamp`.** Carries the data, but pruning stale
   entries means reading *all* fields and filtering in application code —
   O(total) work on every request.
4. **A SORTED SET of `uid` scored by last-seen unix time.** The score *is*
   the timestamp, so "expire the stale" becomes a single range deletion by
   score, and "freshest N first" becomes a single reverse range read. This is
   the canonical answer, and it's what Surwill uses:

```
ZADD    landing:presence  <now> <user_id>          # upsert my heartbeat
ZREMRANGEBYSCORE landing:presence -inf <now-120>   # evict everyone stale
ZREVRANGE landing:presence 0 499                   # freshest first
```

Three commands, each O(log N)-ish, one key, zero background jobs. The
**pruning is lazy**: it happens *inside the read path*, so the set is cleaned
exactly as often as anyone cares about its contents. No cron, no reaper task
— traffic itself is the garbage collector. (Corner case worth knowing: if
traffic stops entirely, the last stale members sit in the set — invisible to
users, bounded in size, and evicted by the very next request. Harmless.)

### Why polling (and not WebSockets/SSE)

Presence *feels* like a real-time feature, so juniors reach for WebSockets.
Run the numbers first: the display updates every 45 s, the payload is ~600
bytes, and the audience is a landing page. A WebSocket buys you sub-second
latency at the cost of a stateful connection per visitor, sticky-session or
pub/sub fan-out across API replicas, reconnect/backoff logic, and a second
code path for auth. **Polling at 45 s is one ordinary HTTP request that
reuses every piece of infrastructure you already have** — auth header, load
balancer, logging, rate limiting. The engineering rule: *push when latency
requirements are below your acceptable polling interval or when the server
must initiate; poll otherwise.* A 45-second-fresh counter is the poster child
for polling.

There's a subtle elegance in Surwill's version: the heartbeat **is** the read.
One POST both registers "I'm here" and returns the current snapshot — the
write and the read share a round trip, so presence costs exactly one request
per user per interval, total.

### Clock discipline

Scores are set from **server time** (`time.time()` in the handler) — never
from anything the client sends. Client clocks are lies (skewed, manually set,
time-zoned); accepting them would let one bad clock instantly expire everyone
or make a member immortal. With a single Redis and server-generated
timestamps, all scores come from the API hosts' clocks — NTP-synced within
milliseconds, which is noise against a 120 s TTL. (Kleppmann's chapter on
unreliable clocks is the deep version of this instinct — see the reading
list.)

## Official documentation quotes

> "A Redis sorted set is a collection of unique strings (members) ordered by
> an associated score."
> — Redis docs, *Sorted sets*, https://redis.io/docs/latest/develop/data-types/sorted-sets/

> "Adds all the specified members with the specified scores to the sorted set
> stored at key. It is possible to specify multiple score / member pairs. If
> a specified member is already a member of the sorted set, the score is
> updated and the element reinserted at the right position to ensure the
> correct ordering."
> — Redis docs, *ZADD*, https://redis.io/docs/latest/commands/zadd/

> "Removes all elements in the sorted set stored at key with a score between
> min and max (inclusive)."
> — Redis docs, *ZREMRANGEBYSCORE*, https://redis.io/docs/latest/commands/zremrangebyscore/

> "As of Redis version 6.2.0, this command is regarded as deprecated. It can
> be replaced by ZRANGE with the REV argument when migrating or writing new
> code."
> — Redis docs, *ZREVRANGE*, https://redis.io/docs/latest/commands/zrevrange/

(That last one matters below — the client library still exposes
`zrevrange()`, but know the modern spelling: `ZRANGE key 0 499 REV`.)

## The code, explained

The feature is a loop through four files — follow one heartbeat end to end.

### 1. The client loop — `surwill-frontend/app/page.tsx`

```tsx
// Live presence strip in the grid header: real profile pictures of logged-in
// users on this page right now (heartbeat → Redis). The viewer leads the row.
const [presence, setPresence] = useState<PresenceSnapshot | null>(null)
useEffect(() => {
  let stopped = false
  const beat = () =>
    landing.presenceHeartbeat()
      .then((s) => { if (!stopped) setPresence(s) })
      .catch(() => { /* non-fatal */ })
  beat()
  const timer = setInterval(beat, 45_000)
  return () => { stopped = true; clearInterval(timer) }
}, [isLoggedIn])
```

Small block, five deliberate choices:

1. **`beat()` runs immediately, then every 45 s.** Without the leading call
   you'd stare at an empty strip for the first 45 seconds.
2. **The `stopped` flag** guards against the classic React race: the user
   navigates away, the component unmounts, and a heartbeat that was already
   in flight resolves afterwards — without the flag it would call `setState`
   on an unmounted component. `clearInterval` stops *future* beats; `stopped`
   neutralizes the *in-flight* one.
3. **`catch(() => {})`** — presence is decoration. If the API hiccups, the
   strip keeps showing the last snapshot rather than erroring the page.
4. **`[isLoggedIn]` as the dependency** — the moment auth state flips, the
   effect tears down and re-runs, so a user who just logged in appears in the
   strip *immediately* (next `beat()` carries their token) instead of up to
   45 s later.
5. **45 000 ms against a 120 s server TTL** — the 2.7× headroom absorbs one
   lost request *and* browser timer throttling: background tabs get their
   `setInterval` clamped (to ≥1 s always, and after 5 minutes Chrome throttles
   further), so an abandoned tab will eventually miss the TTL window and the
   user drops off the strip. For presence that's arguably correct behavior —
   a tab you haven't looked at in ten minutes isn't "on the page" in any
   meaningful sense.

### 2. The API client — `surwill-frontend/lib/api.ts`

```ts
export type PresenceUser = {
  id: string
  first_name: string
  last_initial: string
  profile_photo_url: string | null
}

export type PresenceSnapshot = {
  online_total: number
  users: PresenceUser[]
}

export const landing = {
  presenceHeartbeat: () =>
    request<PresenceSnapshot>('/landing/presence', { method: 'POST' }),
  ...
}
```

`request()` attaches `Authorization: Bearer <access_token>` from
`localStorage` *when a token exists* and sends nothing otherwise — which is
exactly what lets one endpoint serve both audiences (next section). Note it's
a **POST**: the call has a side effect (it registers you), so GET semantics —
cacheable, prefetchable, replayable by proxies — would be wrong. A CDN or
browser cache serving a stale GET response would freeze the strip; worse, a
prefetcher could register presence for a page the user never opened.

### 3. The route + the optional-auth trick — `surwill-backend/app/landing/routes.py` and `app/deps.py`

```python
@router.post("/landing/presence", response_model=PresenceSnapshotResponse, tags=["landing"])
async def presence_heartbeat(
    user_id: OptionalCurrentUserID,
    db: DBSession,
    redis: Redis,
) -> PresenceSnapshotResponse:
    """Landing-page heartbeat: registers logged-in callers as present and returns
    who is on the page right now (first 6 avatars + total logged-in count)."""
    return await services.presence_heartbeat(redis, db, user_id)
```

The interesting dependency is `OptionalCurrentUserID`:

```python
optional_bearer_scheme = HTTPBearer(auto_error=False)

async def get_optional_user_id(
    credentials: Annotated[HTTPAuthorizationCredentials | None, Depends(optional_bearer_scheme)],
) -> str | None:
    if credentials is None:
        return None
    payload = decode_access_token(credentials.credentials)
    if payload is None:
        return None
    return payload.get("sub")

OptionalCurrentUserID = Annotated[str | None, Depends(get_optional_user_id)]
```

Contrast with the strict `get_current_user_id` from lesson 08, which raises
`401` on any problem. Here **every failure path degrades to `None`**:

- `HTTPBearer(auto_error=False)` — no `Authorization` header? FastAPI injects
  `None` instead of rejecting the request.
- An *expired or garbage* token also becomes `None` rather than a 401.

That last point is a real design decision: an anonymous visitor and a visitor
with a stale token get the identical experience (they see the strip, they
aren't counted), and the landing page — the most public URL in the product —
never bounces anyone with an auth error because of a token they forgot they
had. The general pattern to remember: **public endpoints that personalize
when possible want optional auth that fails soft, not strict auth that fails
loud.**

Also note `Redis = Annotated[aioredis.Redis, Depends(get_redis)]` hands every
request the same lazily-created, process-global connection pool — and in dev
or tests, `REDIS_URL=fakeredis://` swaps in an in-memory fake with the same
API, so this whole feature runs without a Redis server (lesson 05's
"dependencies are seams" point, paying rent again).

### 4. The service — `surwill-backend/app/landing/services.py`

```python
# The landing page sends a heartbeat every ~45s. Only LOGGED-IN users are
# tracked and counted: each lives in one Redis sorted set keyed by user id and
# scored by last-seen time, and is considered gone once the heartbeat is older
# than the TTL. Anonymous visitors still receive the snapshot but are neither
# stored nor counted.

_PRESENCE_KEY = "landing:presence"
_PRESENCE_TTL_SECONDS = 120
_PRESENCE_MAX_AVATARS = 6


async def presence_heartbeat(
    redis: aioredis.Redis,
    db: AsyncSession,
    user_id: str | None,
) -> PresenceSnapshotResponse:
    from app.users.models import User

    now = time.time()

    if user_id:
        await redis.zadd(_PRESENCE_KEY, {user_id: now})
    await redis.zremrangebyscore(_PRESENCE_KEY, "-inf", now - _PRESENCE_TTL_SECONDS)
```

Walk it in order:

- **`now = time.time()`** — server clock, captured once so the write and the
  eviction threshold agree on what "now" means within this request.
- **`zadd(key, {user_id: now})`** — an *upsert*: first heartbeat inserts the
  member, every later one just overwrites the score. Last write wins, which
  for "when did we last hear from this user" is precisely the semantics you
  want — two racing heartbeats from two tabs of the same user differ by
  milliseconds and either value is fine. O(log N).
- **`zremrangebyscore(key, "-inf", now - 120)`** — the lazy garbage
  collection: everyone whose last heartbeat is older than the TTL is deleted
  *now*, as a side effect of this request. O(log N + M) where M is the number
  evicted. Because it runs unconditionally (even for anonymous callers), any
  traffic at all keeps the set clean.

```python
    # Most-recent first; validate against the DB so deactivated/deleted accounts
    # are neither shown nor counted and the "+N" math always adds up.
    members = await redis.zrevrange(_PRESENCE_KEY, 0, 499)
    present_ids: list[uuid.UUID] = []
    for m in members:
        try:
            present_ids.append(uuid.UUID(m))
        except ValueError:
            continue
```

- **`zrevrange(key, 0, 499)`** — highest score first, i.e. *freshest first*,
  capped at 500. The cap is a defensive bound: the response work below is
  O(members read), so a pathological or malicious flood of member ids can
  never make this endpoint read an unbounded set. (Works because the client
  library returns `str` — the pool was created with `decode_responses=True`;
  without that flag you'd get `bytes` and `uuid.UUID(m)` would raise on every
  member. A classic integration gotcha.)
- **The `try/uuid.UUID/except` filter** — never trust even your own cache:
  if a junk member ever lands in the set (a bug, a manual redis-cli poke), it
  gets skipped instead of 500-ing the landing page forever.

```python
    ordered: list = []
    if present_ids:
        result = await db.execute(
            select(User).where(
                User.id.in_(present_ids),
                User.is_active == True,  # noqa: E712
                User.deleted_at.is_(None),
            )
        )
        by_id = {u.id: u for u in result.scalars()}
        ordered = [by_id[i] for i in present_ids if i in by_id]
        # The requesting user sees themselves leading the row.
        if user_id:
            me = uuid.UUID(user_id)
            ordered.sort(key=lambda u: u.id != me)
```

Three points of craft here:

1. **Redis is the index, Postgres is the truth.** The sorted set answers
   *who was here recently*; the DB answers *who still deserves to be shown*.
   A user deactivated or soft-deleted five minutes ago might still have a
   live heartbeat in Redis — the `is_active` / `deleted_at` filter guarantees
   they are neither displayed nor counted. This is the standard division of
   labor whenever a cache indexes durable entities: filter through the source
   of truth on the way out.
2. **`ordered = [by_id[i] for i in present_ids if i in by_id]`** — the SQL
   `IN (...)` clause returns rows in *arbitrary* order, so the code re-imposes
   Redis's recency order by walking `present_ids` (already sorted) and
   picking from the dict. Forgetting this re-ordering step is one of the most
   common "why is my list shuffled" bugs with `IN` queries.
3. **`ordered.sort(key=lambda u: u.id != me)`** — a lovely Python idiom: the
   key maps *me* to `False` (0) and everyone else to `True` (1), and because
   Python's Timsort is **stable**, this moves the viewer to the front while
   preserving recency order among the rest. One line, no special-casing.

```python
    return PresenceSnapshotResponse(
        online_total=len(ordered),
        users=[
            PresenceUser(
                id=u.id,
                first_name=u.first_name,
                last_initial=(u.last_name[:1].upper() if u.last_name else ""),
                profile_photo_url=u.profile_photo_url,
            )
            for u in ordered[:_PRESENCE_MAX_AVATARS]
        ],
    )
```

- **`online_total` is computed *after* DB validation**, and the avatar list
  is a slice of the same `ordered` list — so the UI's overflow tile
  `+{online_total - users.length}` can never go negative or lie. Deriving
  both numbers from one list is what makes the math *provably* consistent;
  computing the count from `ZCARD` and the avatars from the DB would let them
  drift.
- **Data minimization by construction:** the wire format is first name, one
  initial, and an avatar URL. No username, no email, no id-to-profile lookup
  surface for scrapers watching the world's most public endpoint. When you
  expose "who is here" publicly, deciding *how much of who* is a security
  decision, and the schema is where you enforce it:

```python
class PresenceUser(BaseModel):
    id: uuid.UUID
    first_name: str
    last_initial: str
    profile_photo_url: str | None

class PresenceSnapshotResponse(BaseModel):
    online_total: int
    users: list[PresenceUser]
```

### 5. Rendering — the avatar strip

Back in `app/page.tsx`, the snapshot renders as overlapping tiles: each
avatar is absolutely stacked with a negative margin (`-ml-3.5 md:-ml-6`) and
a *descending* `zIndex: presence.users.length - i` so earlier (fresher,
viewer-first) faces sit on top of later ones; hover lifts a tile
(`hover:-translate-y-1.5 hover:z-30`) and reveals a name tooltip; and the
overflow tile renders `+{presence.online_total - presence.users.length}` only
when the total exceeds the six avatars sent. The strip renders nothing at all
until the first snapshot arrives *and* `online_total > 0` — an empty "0
people here" is worse than absence, a small but real product judgment.

## How this is used in production

- **Startups** ship exactly this shape: heartbeat + Redis TTL'd structure +
  poll, because it's an afternoon of work and rides existing infra. The
  moment it exists it does double duty as *social proof* (Surwill's actual
  motive — a live strip of real faces converts better than any testimonial)
  and as a crude concurrent-users metric for the founders' dashboard.
- **Document collaborators / "green dot" systems** (Google Docs, Slack,
  Figma, LinkedIn-style presence) run the same logical model — heartbeat,
  TTL, last-write-wins — but push updates over the WebSocket they *already
  hold open* for the core product (edits, messages). The rule generalizes:
  presence piggybacks on whatever channel exists; only the delivery changes,
  never the heartbeat/TTL core.
- **Ride-hailing / delivery** (Uber-scale) is presence with a geo dimension:
  driver phones heartbeat location every few seconds into in-memory
  geo-indexed stores; a driver whose heartbeats stop is unmatchable within a
  TTL. Same silence-means-absence contract, hundreds of thousands of members,
  sharded by city.
- **Counters at scale**: sites showing "12,431 people viewing" (ticketing,
  flash sales) typically stop tracking *identities* and track only a number —
  sharded counters or HyperLogLog approximations — because the roster itself
  stops being renderable. Knowing where the identity-roster design gives way
  to the pure-counter design is a scaling-stage answer interviewers love.
- **Monitoring**: whatever the scale, the operational metrics are the same —
  sorted-set cardinality (`ZCARD`) as a gauge, heartbeat endpoint p99, and
  Redis memory. Presence is also the classic *victim* of an outage-adjacent
  stampede: when the API comes back after downtime, every open tab's next
  tick lands at once; the fixed poll interval plus the cheapness of the
  endpoint is what makes that survivable.

## Advanced corner

**Atomicity — and why its absence is fine here.** The handler issues ZADD,
ZREMRANGEBYSCORE, and ZREVRANGE as three separate round trips. Two concurrent
requests can interleave between them — one request might read a member that
another is about to evict, so a snapshot can be stale by one request's worth
of time. That's invisible next to the 45 s poll interval, so correctness
doesn't need atomicity. *Latency* might: three awaited round trips at ~0.5 ms
each is nothing on localhost but adds up through a TLS-terminated managed
Redis. The upgrade path is a **pipeline** (one round trip, still not atomic),
then **MULTI/EXEC** (atomic, no intermediate reads needed here), then a **Lua
script / Redis function** (atomic *and* can compute the answer server-side —
e.g., return only members whose score passes the threshold, skipping the
eviction write entirely). Know the ladder and when each rung is worth it.

**Complexity budget.** Per heartbeat: ZADD O(log N), ZREMRANGEBYSCORE
O(log N + M), ZREVRANGE O(log N + 500), one `SELECT ... WHERE id IN (≤500)`
on a primary key. With N in the thousands this is microseconds of Redis time;
the DB query is the expensive part, which leads to…

**Scaling the read path.** The current code hits Postgres with an `IN (≤500)`
on *every heartbeat from every visitor*. At Surwill's scale, fine. At 10k
concurrent visitors polling every 45 s (~220 req/s), you'd fix it in stages:
(1) cache the *rendered snapshot* in Redis for 10–20 s — presence tolerates
staleness by definition, so all readers share one DB query per window;
(2) store display data (name, avatar URL) *in* the member payload as JSON so
the DB drops out of the hot path, accepting staleness of profile edits;
(3) split the write (`ZADD`, per user) from the read (cached snapshot,
global) into separate endpoints so anonymous visitors stop triggering writes
altogether. Being able to narrate these stages *in order of need* is what
"senior" sounds like.

**Horizontal scale is already solved** — notice you get it for free: any
number of API replicas behind a load balancer share the one sorted set in
Redis. No sticky sessions, no state on the API host. This is the quiet payoff
of pushing ephemeral state into a shared store, and the exact thing that
becomes *hard* if you move to WebSockets (now each replica holds live
connections and needs pub/sub to fan updates out to peers' clients).

**Failure modes.** Redis down → `get_redis()`'s pool raises, the endpoint
500s, the frontend's `catch(() => {})` swallows it, the strip freezes on its
last snapshot; users lose a vanity widget, nothing else. Redis restarts empty
→ within one heartbeat interval the set repopulates. Worker/API clock skew →
all writers are NTP-synced servers; a second of skew against a 120 s TTL is
noise (contrast: it would *not* be noise for a 2 s TTL lock — TTL-based
designs inherit clock assumptions, always state them). A user opens five tabs
→ five heartbeats upsert the same member: sorted sets deduplicate by member
string, so the count stays honest. Two *different* users behind one browser
profile can't collide because the member is the user id from the JWT, not a
session id.

**What most people get wrong at this level:** per-member expiry ("just set a
TTL on each member" — doesn't exist inside one Redis key; the score *is* the
TTL mechanism), computing the total from `ZCARD` while filtering avatars
through the DB (drifting `+N` math), trusting client timestamps, GET instead
of POST for a side-effecting heartbeat, and reaching for WebSockets before
doing the latency-requirement arithmetic.

## Best practices

- **Server assigns all timestamps.** The client asserts presence; it never
  gets to say *when*.
- **TTL ≥ 2× heartbeat interval** (plus jitter allowance). One lost packet
  must not flicker a user offline.
- **Prune lazily in the read path** for small/medium sets — traffic is the
  garbage collector, and there's no cron to forget, break, or monitor.
- **Validate cache-indexed entities against the source of truth on the way
  out** (`is_active`, `deleted_at`). Redis remembers people Postgres has
  disowned.
- **Derive the count and the roster from the same filtered list** so overflow
  math can't lie.
- **Minimize the public payload** — first name + initial + avatar is a
  deliberate ceiling, not laziness.
- **Fail soft on both ends**: optional auth that degrades to anonymous;
  a client `catch` that keeps the last good snapshot.
- **Side effects ⇒ POST.** Heartbeats mutate state; keep HTTP semantics
  honest so caches and prefetchers can't hurt you.

## Common mistakes & gotchas

- **`decode_responses` mismatch.** Pool created without `decode_responses=True`
  returns `bytes`; `uuid.UUID(b"...")` raises `AttributeError`-adjacent chaos.
  Surwill sets it at pool creation in `get_redis()` — if you ever add a second
  pool, match it.
- **`IN` queries shuffle your order.** SQL gives no ordering guarantee for
  `WHERE id IN (...)`; re-impose the cache's order in application code (the
  `by_id` dict walk).
- **Heartbeat effect without cleanup** — a `setInterval` that survives
  unmount keeps polling forever; the `stopped` flag *and* `clearInterval`
  are both required (they cover different races).
- **Background-tab throttling** will stretch your interval; treat it as a
  liveness *feature* for presence, but never build anything that *requires*
  a background tab to tick on time.
- **Unbounded reads**: no cap on `zrevrange` means an adversary who can get
  members into your set controls your response size. Cap it (500 here).
- **Testing with real time**: presence logic is time-parametric — in tests,
  inject the clock or manipulate scores directly (`ZADD` with an old
  timestamp) rather than `sleep(121)`. With `fakeredis://` this is trivial.
- **Forgetting the deprecation**: `ZREVRANGE` is deprecated since Redis 6.2 —
  fine via the client method today, but write `ZRANGE ... REV` in new
  raw/Lua code and say so in interviews.

## Where AI helps (and where it hurts)

**Helps:** generating the boilerplate around a pattern you've already chosen
(the Pydantic schemas, the React effect with correct cleanup — which AI
usually gets right *if you ask for the cleanup explicitly*); translating the
three-command Redis sequence into a Lua script; enumerating failure modes as
a checklist you then verify; converting this exact feature between stacks
(e.g., "same design with Django + channels").

**Hurts:** the *store and protocol choice* — asked cold for "show online
users", models frequently over-engineer straight to WebSockets + pub/sub, or
under-engineer into a Postgres `last_seen` column with a cron. Both are
defensible somewhere; neither is justified *by your numbers* unless you do
the arithmetic yourself. Also risky: AI-suggested Redis snippets that quietly
assume a different `decode_responses` setting or use per-key `EXPIRE` on
members (which doesn't exist) — run against fakeredis before believing.

## Learn independently

- **Kleppmann, *Designing Data-Intensive Applications*** — Ch. 8 "The Trouble
  with Distributed Systems" (unreliable clocks, process pauses — the deep
  version of "never trust client time; TTLs inherit clock assumptions") and
  Ch. 11's discussion of derived, rebuildable state (presence is the purest
  possible example of state you can afford to lose).
- **Carlson, *Redis in Action*** — Ch. 2 (login/session cookies with sorted
  sets — nearly this exact pattern) and Ch. 3 §3.5 (sorted-set commands);
  Ch. 6's counters/semaphores for where the pattern grows next. Free online
  via Redis's site.
- **Grigorik, *High Performance Browser Networking*** (free at hpbn.co) —
  Ch. 15–17 (XHR polling, SSE, WebSocket): the definitive treatment of the
  poll-vs-push decision this lesson makes informally.
- **Primary docs:** Redis sorted sets
  (https://redis.io/docs/latest/develop/data-types/sorted-sets/), `ZADD` /
  `ZREMRANGEBYSCORE` / `ZRANGE` command pages; FastAPI security dependencies
  (https://fastapi.tiangolo.com/tutorial/security/) for `auto_error=False`;
  MDN `setInterval` — see the "timeouts in inactive tabs" notes for the
  throttling behavior cited above
  (https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout#timeouts_in_inactive_tabs).

## Interview preparation

**Q1. Design "show who's online" for a web app. Walk me through it.**
*Model answer:* Presence is ephemeral, loss-tolerant state, so it goes in an
in-memory store, not the primary DB. Clients heartbeat every H seconds
(silence means absence — clients can't reliably say goodbye); the server
stores `user → last_seen` in a Redis sorted set scored by server time, prunes
members older than TTL T (T ≥ 2H) with one `ZREMRANGEBYSCORE` inside the read
path, and returns the freshest N via `ZRANGE ... REV`. Read staleness is
bounded by H + T, which the product tolerates by definition. Poll rather than
push while acceptable latency exceeds the poll interval; the design scales
horizontally for free because API replicas share the Redis key. I'd validate
members against the user table on the way out so deactivated accounts vanish
immediately, and I'd cap both the range read and the response payload.

**Q2. Why a sorted set? Why not a SET, or one key per user with EXPIRE?**
*Model answer:* Redis has no per-member TTL within a key, so a plain SET
can't expire individuals. One key per user gives free expiry but makes
enumeration terrible — SCAN over the keyspace or a secondary index that
itself needs cleanup. A sorted set makes the timestamp the score, so
expiration is a range delete and "freshest N" is a range read — both
logarithmic, both on one key. It's the data structure where *both* required
queries are native.

**Q3. Your three Redis commands aren't atomic. Is that a bug?**
*Model answer:* No — interleaving can only make a snapshot stale by one
request, invisible under a 45 s poll. Presence needs bounded staleness, not
linearizability. If round-trip latency mattered I'd pipeline; if I later
needed read-your-write precision I'd move the sequence into a Lua script,
which also lets me filter by score server-side instead of evicting. The
skill is matching the consistency spend to the requirement, not maximizing
it.

**Q4. The user count and the avatars must never contradict (no "+-2 others").
How do you guarantee that?**
*Model answer:* Derive both from a single list: fetch members, filter through
the DB (active, not deleted), then `total = len(list)` and
`avatars = list[:6]`. The failure smell is computing the total from one
source (`ZCARD`) and the faces from another (post-filter DB rows) — any
filter divergence shows up as impossible math in the UI.

**Q5. When does this design break, and what's the next architecture?**
*Model answer:* Three independent walls. (1) *Read amplification*: every
visitor's heartbeat triggers a DB validation query → cache the rendered
snapshot for 10–20 s, or denormalize display data into the member payload.
(2) *Latency requirement drops* (chat-style "typing…" indicators) → move
delivery to an existing WebSocket/SSE channel, keep the heartbeat+TTL core,
add pub/sub so all API replicas can push. (3) *Roster stops being
renderable* (tens of thousands) → stop tracking identity for display, keep a
counter (sharded counters or HyperLogLog if approximate is fine) and show
"12.4k online". Each is an incremental change because the heartbeat/TTL
contract stays constant.

**Q6. Why does the endpoint accept anonymous callers at all?**
*Model answer:* The snapshot is the landing page's social proof — its whole
job is to be visible to logged-*out* visitors deciding whether to sign up.
Optional auth (`HTTPBearer(auto_error=False)`, invalid tokens degrade to
`None`) lets one endpoint serve both: authenticated callers write + read,
anonymous callers only read. Failing soft also means an expired token never
turns the most public page in the product into a 401.

**Q7. A user closes their laptop. Describe exactly what the system does.**
*Model answer:* Nothing, immediately — that's the point. Heartbeats stop; the
user's score freezes. For up to 120 s they still appear (ghost time). The
first heartbeat from *anyone* after their score passes `now − 120` evicts
them via `ZREMRANGEBYSCORE`. If traffic is zero, the stale member sits
invisible in the set until the next request — bounded garbage, no reaper
needed. Total detection latency: TTL, worst case TTL + gap to next request.

## Exercises against the codebase

**Easy — expose the count everywhere.** Add a `GET /landing/presence/count`
endpoint returning `{count: int}` computed *without touching Postgres*
(prune, then `ZCARD`). Acceptance: no `AsyncSession` dependency; document in
the docstring why this count may briefly exceed the strip's `online_total`
(no DB validation) and why that's acceptable for a navbar badge.

**Medium — per-page presence.** Generalize the service to
`presence_heartbeat(redis, db, user_id, page: str)` using keys
`presence:{page}` with `page` validated against an allowlist
(`landing`, `search`, `profile`). Frontend: heartbeat with the current page.
Acceptance: two browsers on different pages never see each other; the
allowlist rejects arbitrary key creation (why does an unvalidated `page`
parameter hand strangers write access to your Redis keyspace?).

**Medium — one round trip.** Rewrite the three Redis calls as a single
pipeline, then measure both variants 1 000× against real Redis (not fake)
with `time.perf_counter()`. Acceptance: a comment in the code with your
measured numbers and a one-paragraph verdict on whether the pipeline is
worth it at Surwill's scale.

**Hard — kill the DB from the hot path.** Store a JSON payload
(`{"fn": ..., "li": ..., "ph": ...}`) as the member (or in a parallel HASH),
written on heartbeat from the authenticated user's own profile. Serve
snapshots without the `SELECT`. Then answer in writing: how do deactivated
users disappear now (you removed the filter!), and how stale can a renamed
user's display be? Acceptance: a failing-then-passing test that deactivates a
user and asserts they vanish within one TTL window — using fakeredis and
injected timestamps, no `sleep`.

**Hard — push it.** Replace the client's `setInterval` *read* with an SSE
stream (`EventSource` + a FastAPI `StreamingResponse` that publishes a new
snapshot every 10 s or on change), keeping the POST heartbeat for writes.
Acceptance: works through the Docker compose setup; a README note on what
now breaks when you scale the API to two replicas, and the pub/sub shape
that fixes it.

## Key takeaways

- Presence is **ephemeral, loss-tolerant, bounded-staleness** state — the
  profile that says Redis, not Postgres, and polling, not WebSockets (until
  the latency arithmetic says otherwise).
- **Heartbeat + TTL** with `TTL ≥ 2× interval` is the universal presence
  contract; silence means absence because clients can't reliably say goodbye.
- A **sorted set scored by last-seen time** makes both required operations
  native: expire = range-delete by score, roster = reverse range read; the
  pruning rides the read path, so traffic is the garbage collector.
- **Redis is the index, Postgres is the truth**: validate members against
  `is_active`/`deleted_at` on the way out, and derive count + avatars from
  the *same* filtered list so the `+N` math cannot lie.
- **Optional auth that fails soft** (`auto_error=False`, bad token → `None`)
  is the right shape for public endpoints that personalize opportunistically.
- Server time only; POST because it mutates; cap every unbounded read; send
  the minimum identity the UI needs — each is one line of code and one
  sentence in an interview, and together they're the difference between "I
  used Redis" and "I designed a presence system."
