# FastAPI from a Real Codebase — the Surwill Track

Learn production FastAPI by dissecting **Surwill**, a real Georgian-first gift-wishlist
platform: FastAPI + async SQLAlchemy + PostgreSQL + Redis + arq workers + a
five-strategy web-scraping subsystem + a pytest suite with golden-file tests,
all shipped with Docker and GitHub Actions.

This track has **two explicit goals**:

1. **Master FastAPI** — not toy-tutorial FastAPI, but the framework as it is
   actually used: layered architecture, dependency injection, async database
   access, background jobs, auth, testing, deployment.
2. **Own this project in an interview** — every lesson ends with interview Q&A,
   and lesson 13 is a full playbook for presenting Surwill as *your* system:
   architecture narrative, deep-dive answers, and the war stories that make an
   interviewer believe you actually built and operated it.

## How this track differs from the others

The `python/`, `sql/`, `linux/` tracks teach concepts with constructed examples.
This track inverts that: **every example is real code from Surwill**, quoted with
its file path, and explained line by line — theory is introduced exactly where the
codebase needs it. Lessons keep the repo's standard sections (theory, official-doc
quotes, production usage, advanced corner, best practices, gotchas, AI guidance,
book references, interview Q&A, key takeaways), but "Real-world examples" becomes
**"The code, explained"**, and homework becomes **"Exercises against the codebase"**
— things to build or break in Surwill itself.

> The code snapshots in these lessons were taken from the Surwill backend as of
> July 2026 (the commit that replaced phone/OTP auth with email auth and added the
> production-hardening batch). If the live repo has drifted, the explanations still
> hold — the snapshot is the teaching artifact.

## The system in one paragraph

Surwill lets users build gift wishlists. Items can be added by pasting a store
URL — a **scraping pipeline** (Shopify JSON → httpx → curl_cffi → Playwright →
managed API, then a 5-layer HTML extractor) turns the URL into a titled, priced,
imaged product. Friends **reserve** items (atomically, with surprise-preservation
rules), an **arq worker** refreshes prices/stock hourly and sends notifications
on price drops, and the whole thing runs as five containers: Postgres, Redis,
API, worker, Next.js frontend.

## Syllabus

| # | Lesson | Status |
|---|--------|--------|
| 01 | [Architecture & project layout](lesson-01-architecture-and-project-layout.md) | ✅ |
| 02 | [ASGI, uvicorn & the application object](lesson-02-asgi-uvicorn-and-the-application-object.md) | ✅ |
| 03 | [Routing, requests & responses](lesson-03-routing-requests-and-responses.md) | ✅ |
| 04 | [Pydantic: validation, schemas & settings](lesson-04-pydantic-validation-and-settings.md) | ✅ |
| 05 | [Dependency injection](lesson-05-dependency-injection.md) | ✅ |
| 06 | [Async SQLAlchemy & the data layer](lesson-06-async-sqlalchemy-and-the-data-layer.md) | ✅ |
| 07 | [Alembic migrations](lesson-07-alembic-migrations.md) | ✅ |
| 08 | [Auth: JWT, refresh rotation & API security](lesson-08-auth-jwt-and-api-security.md) | ✅ |
| 09 | [**Deep dive:** background jobs with arq](lesson-09-background-jobs-arq-workers.md) | ✅ |
| 10 | [**Deep dive:** the scraping subsystem](lesson-10-the-scraping-subsystem.md) | ✅ |
| 11 | [**Deep dive:** testing with pytest](lesson-11-testing-with-pytest.md) | ✅ |
| 12 | [Docker, deployment & CI](lesson-12-docker-deployment-and-ci.md) | ✅ |
| 13 | [The interview playbook](lesson-13-interview-playbook.md) | ✅ |
| 14 | [**Deep dive:** live presence — Redis sorted sets & heartbeats](lesson-14-live-presence-redis-sorted-sets.md) | ✅ |

Lessons 09–11 are the requested deep dives — noticeably longer and more granular
than the rest. Do them slowly, with the Surwill repo open in the other window.
Lesson 14 dissects one small real feature (the landing page's live "on this
page right now" strip) end to end — frontend heartbeat → optional-auth route →
Redis sorted set — as a compressed system-design case study.

## How to study

1. Have the Surwill backend open side by side; every snippet cites its file.
2. After each lesson, do the **Exercises against the codebase** — reading about
   `Depends` is nothing; adding a new dependency and watching a test override it
   is everything.
3. After lesson 13, rehearse the interview narrative **out loud**, twice.
