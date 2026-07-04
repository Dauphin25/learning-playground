# Lesson 13 — The Interview Playbook

**Track:** FastAPI (Surwill) · **Time:** ~2.5h + rehearsal · **Prerequisites:** lessons 01–12
**Objectives:** turn everything you've studied into a performance — a crisp architecture narrative, three deep-dive stories you can tell on demand, ready answers to the predictable follow-ups, and honest coverage of the system's weaknesses.

---

## Why this matters

You can know the code cold and still interview poorly if you can't *sequence*
it. This lesson is the delivery layer: the 60-second pitch, the deep dives an
interviewer will pull you into, the war stories that prove you operated the
thing, and the weaknesses you should raise before they do. Rehearse these out
loud — reading is not practice.

## Part 1 — The 60-second architecture pitch

Memorize the shape, not the words. Deliver in one breath, then stop and let them
steer.

> "Surwill is a gift-wishlist platform — think a shared registry you build by
> pasting product links. It's a modular-monolith FastAPI backend over PostgreSQL
> and Redis, with a separate arq worker and a Next.js frontend — five containers.
> The codebase is sliced vertically by domain — auth, boards, items,
> reservations — each with its own models, schemas, routes, and thin service
> layer. Three parts are interesting to talk about: the item-import **scraping
> pipeline**, which escalates through five fetch strategies from a cheap JSON
> probe up to headless Chromium and a managed API; the **reservation flow**,
> which uses Postgres row locking so a gift can't be double-claimed; and the
> **auth system**, short-lived JWTs plus rotating refresh tokens. The worker
> handles everything slow or scheduled — email, hourly price refresh, cleanup —
> and Redis is the task queue, the scrape cache, and the rate limiter."

That paragraph names your three deep dives *and* invites the interviewer to pick
one. You're steering while sounding open.

## Part 2 — The three deep dives

For each: a one-line hook, the 2-minute telling, and the follow-ups you'll get.

### Deep dive A — The scraping pipeline (your strongest)

**Hook:** "Turning an arbitrary store URL into a titled, priced, imaged product,
across a dozen store platforms, some behind Cloudflare."

**The telling (2 min):** cost-ordered escalation. Shopify JSON probe (free,
perfect when it hits) → httpx with rotated browser headers, SSRF checks,
per-domain throttling, and 429/backoff → on a 403, curl_cffi to reproduce
Chrome's TLS/JA3 fingerprint (a 403 is usually TLS fingerprinting, not
authorization) → for JS-only SPAs, headless Chromium with stealth patches →
last resort, a managed scraping API, off unless configured. Every stage only
fills missing fields and stamps how it got the data. The HTML itself goes
through a five-layer extractor — platform JSON, JSON-LD, Open Graph, microdata,
HTML heuristics — best source first. Because the extractor is 2600 lines of
fuzzy logic, its correctness is pinned by golden-file tests against real store
HTML.

**Follow-ups you must have ready:**
- *SSRF?* → resolve-and-block private/loopback/link-local (the 169.254 metadata
  endpoint), re-checked on every redirect hop and in the managed path too.
- *Why not just Playwright?* → cost; it's a whole browser, so it's the fourth
  rung, not the first.
- *New store?* → fixture + golden test; adapter selector only if the test proves
  a gap; never edit the engine.
- *Legal/ethical?* → prefer official APIs, robots flag, no CAPTCHA-solving; it's
  a policy question, not just a technical one.

(Full depth: lesson 10.)

### Deep dive B — The reservation race

**Hook:** "The one invariant we could never violate: two people must not both
think they've reserved the same gift."

**The telling:** `SELECT … FOR UPDATE` locks the item row; a concurrent request
blocks on the lock, and when it proceeds it sees the committed `reserved` status
and gets a 409. The invariant is enforced by Postgres at READ COMMITTED, not by
application checks that can interleave. One transaction, `flush` mid-way to get
the reservation id for the notification, single `commit`. Alternatives I
considered: an optimistic conditional UPDATE checking rowcount, or a partial
unique index — I chose the lock because I need the loaded row for permission
checks in the same transaction.

**Follow-ups:** isolation levels (why READ COMMITTED suffices; what REPEATABLE
READ would change); deadlock avoidance (consistent lock ordering); how it scales
(`FOR UPDATE SKIP LOCKED` for queue-like workloads). (Full depth: lesson 06.)

### Deep dive C — Auth lifecycle

**Hook:** "Cheap stateless request auth, but still able to revoke a session
instantly-ish."

**The telling:** 15-minute HMAC-signed JWT access tokens — any instance verifies
with one hash, no DB. Revocability comes from 30-day refresh tokens stored
*hashed* in Postgres and *rotated* on every use (single-use; reuse is
detectable). Logout and password-reset just revoke refresh rows; the access
token rides out ≤15 minutes — the deliberate revocation-lag trade-off. bcrypt
for passwords (slow, salted) vs SHA-256 for the high-entropy refresh tokens
(fast is fine). Google login verifies the ID token's signature against Google's
JWKS plus audience/issuer/email_verified before linking.

**Follow-ups:** where tokens live (localStorage today — XSS-readable; httpOnly
cookies is the documented migration); HS256 vs RS256 (asymmetric once a second
service must verify); enumeration-proof reset flow. (Full depth: lesson 08.)

## Part 3 — The war stories (proof you operated it)

Anyone can describe architecture; stories about *fixing* things prove you built
and ran it. Have two ready.

**Story 1 — the async event-loop pooling bug (lesson 11).** "Every async DB
test started failing with asyncpg's 'another operation is in progress.' The
cause was that pytest-asyncio ran tests on separate event loops while
SQLAlchemy's pool cached asyncpg connections — and an asyncpg connection is
bound to its creating loop. The fix was `NullPool` on the test engine so each
test opens a fresh connection on its own loop. It's the difference between
bumping a timeout and understanding loop/connection affinity." *Signals:* async
depth, root-cause debugging, reading tracebacks.

**Story 2 — settings vs. the environment (lesson 04).** "Removing the OTP
feature, I deleted its settings fields — and the next deploy crashed, because the
containers still had the old env vars and newer pydantic-settings rejects unknown
keys with `extra_forbidden`. The fix was operational — purge the stale vars — and
the lesson was that a settings class and its deploy environment version
together; deleting a field is a breaking change for the environment." *Signals:*
config-as-contract, ops awareness, reading the error instead of downgrading.

## Part 4 — Rapid-fire predictable questions

Have a crisp sentence for each; expand only if pushed.

- **Why a monolith?** One team, one product, one DB; vertical slicing keeps
  extraction cheap; the scraper is the first thing I'd pull out.
- **Sync vs async here?** I/O-bound (DB, HTTP scraping, Redis) — async lets one
  loop interleave thousands of in-flight requests; blocking calls go to
  `asyncio.to_thread`.
- **N+1?** `lazy="raise"` makes accidental lazy loads crash in dev; list
  endpoints batch-enrich with an IN query.
- **Where's business logic?** Service layer; routes only authenticate, delegate,
  serialize; services raise domain exceptions mapped to status codes.
- **Migrations?** Alembic, expand→migrate→contract for zero downtime; run on
  boot for one replica, release phase when scaling.
- **How do you test the scraper?** Golden files — real HTML frozen as fixtures,
  deterministic field assertions; pure functions for the helpers.
- **Rate limiting?** Redis window counters keyed on IP+identity, enforced in
  production; login and password-reset are the protected surfaces.
- **Secrets?** Typed pydantic-settings from env; boot-time guard refuses prod
  with dev secrets.
- **Background jobs & delivery guarantees?** arq on Redis; at-least-once, so
  handlers are idempotent (gap-fill, re-check state, stamp-before-work).
- **Scaling bottleneck first to hit?** DB connections (pool×workers arithmetic)
  and the Playwright memory footprint of scraping.

## Part 5 — Own the weaknesses (before they find them)

Volunteering limitations reads as senior, not weak. Keep these ready:

- **Tokens in localStorage** — XSS-readable; httpOnly-cookie migration is
  planned. *(I'd add CSP as an interim.)*
- **Playwright in the API image** — bloats it; splitting scraping into its own
  worker image is the first infra optimization.
- **Backend runs as root** — one-line `USER` fix, complicated by Playwright's
  `--no-sandbox`.
- **Migrate-on-boot** — races with multiple replicas; release phase is the fix.
- **No email verification** — a product decision (low-sensitivity data), not an
  oversight; the reset flow is still enumeration-proof.
- **Rate limits production-only** — keeps dev/test frictionless but means staging
  doesn't exercise them.
- **Scraper is an arms race** — anti-bot evolves; the managed-API escape hatch
  exists precisely so we don't maintain a proxy/CAPTCHA farm.

Pattern for each: *name it → why it's acceptable now → what the fix is.* That
triad is what distinguishes "aware" from "defensive."

## Part 6 — System-design extension questions

Interviewers may push past what's built. Bridges from Surwill:

- **"Scale scraping to millions of URLs/day."** Dedicated scraper-worker fleet;
  multiple arq queues by priority; distributed rate limiting (already Redis-
  shared); managed-API for the hard tail; a URL-dedup/idempotency layer;
  possibly a Postgres `SKIP LOCKED` claim queue or Kafka for volume/replay.
- **"Instant session revocation."** Redis denylist keyed by token jti checked
  per request (pay the lookup JWTs avoided), or drop access-token TTL further,
  or a token-version claim compared to the user row.
- **"Multi-region."** Read replicas + a second engine for reads; Redis per region
  with the queue centralized or federated; the SSRF and rate-limit layers are
  already stateless-per-request.
- **"Recommend gifts."** New slice + worker jobs computing features off the
  existing item/reservation data; the vertical-slice architecture makes this
  additive, not invasive.

## Best practices for the interview itself

- Lead with the shape, then let them pick the branch — don't monologue all three deep dives unprompted.
- Draw the five-box diagram while you talk; a whiteboard anchors the conversation.
- When you don't know, say the trade-off you'd investigate — reasoning beats guessing.
- Use precise nouns: JA3, READ COMMITTED, at-least-once, `FOR UPDATE`, JWKS,
  `NullPool`. Precision is the credibility signal.
- Bring a weakness voluntarily; it inverts the power dynamic from interrogation
  to peer discussion.

## Common mistakes

- Memorizing a script and derailing when interrupted — memorize *structure*, improvise words.
- Overclaiming ("exactly-once delivery," "unhackable") — every senior hears it as a red flag.
- Diving into code before framing the system — always zoom out first.
- Hiding weaknesses — they surface anyway, and worse if you looked evasive.

## Where AI helps (and hurts)

AI is a superb rehearsal partner: paste your pitch and have it play a skeptical
staff engineer drilling follow-ups. It's poor at knowing *which* three stories
are yours to tell — that judgment (and the lived war stories) is the part that's
authentically you, and it's what interviews actually test.

## Learn independently

- **Designing Data-Intensive Applications** (Kleppmann) — the vocabulary for
  every scaling follow-up.
- *System Design Interview* (Alex Xu), vols 1–2 — the extension-question genre.
- **Site Reliability Engineering** (Google, free online) — for the ops framing
  of health, rollout, and failure.
- Re-read lessons 06 (reservation), 08 (auth), 09–11 (worker/scraper/tests) —
  your three deep dives and both war stories live there.

## Interview Q&A (meta — about presenting)

**Q: Tell me about a project you're proud of.**
A: Deliver the Part 1 pitch, then: "I can go deep on any of three areas — the
scraping pipeline, the reservation concurrency, or the auth lifecycle — which is
most useful?" *(You've framed the whole conversation in 75 seconds and handed
them the wheel.)*

**Q: What would you do differently?**
A: Pick two from Part 5 with the name→why-acceptable→fix triad. "I'd move tokens
from localStorage to httpOnly cookies — it's XSS-readable today; acceptable for
launch behind a CSP, but the cookie migration is the real fix. And I'd split
Playwright out of the API image — it bloats every API deploy for a feature only
the worker uses."

**Q: What was the hardest part?**
A: The scraper — "not any single technique, but the *orchestration*: deciding
the escalation order by cost, making each stage supplement rather than clobber,
defending the SSRF surface across redirects, and testing a fundamentally fuzzy
system deterministically with golden files. It's the part that taught me the
most about designing for the real, messy world instead of the happy path."

## Exercises (rehearsal)

- **Easy:** Record yourself giving the 60-second pitch; cut it to truly 60s.
- **Medium:** Have someone (or an AI) interrupt each deep dive after 30 seconds
  with a follow-up; practice pivoting without losing the thread.
- **Hard:** Whiteboard the full system from memory in 5 minutes — five boxes,
  data flow, the three deep-dive subsystems marked — while narrating. Do it
  twice; the second time is the one that sticks.

## Key takeaways

- Lead with the 60-second shape; name your three deep dives; let them choose.
- Deep dives = hook + 2-minute telling + rehearsed follow-ups (scraper, reservation, auth).
- Two war stories (NullPool loop bug, settings/env `extra_forbidden`) prove you *operated* it.
- Own weaknesses with name→why-acceptable→fix; precise nouns are the credibility signal.
- Rehearse out loud, structure-not-script, zoom out before zooming in.
