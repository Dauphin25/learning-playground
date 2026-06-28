# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A personal **6-month self-study curriculum** to master **Python, SQL, and Linux** — independently and combined. The owner is **already experienced** (prior Linux, SQL, Python/Django, frontend, DevOps exposure; currently in Application Support) and wants to become a **confident mid-level engineer who can walk into a mid-level interview and pass it without hesitation**, reaching into **senior-flavored depth** on each topic. Target roles: **data engineering, DevOps/platform engineering, and Python backend (FastAPI)**. It is a *content/learning* repository, not an application. The deliverables are lessons, exercises, and capstone project guides written in Markdown.

The full intent is captured in `description.md` — read it before authoring or restructuring content.

## Audience, level & depth (IMPORTANT — read before authoring)

- **This is NOT beginner material.** Assume the reader already programs, knows basic shell/SQL, and can pick up syntax quickly. Do not spend pages on trivia they already know; instead, go *deep*, expose the *why* and the internals, and pull in advanced/edge-case material.
- **Calibration target:** by the end of a track the reader should confidently pass a **mid-level interview** and hold their own on **senior-level follow-ups**. Every lesson should therefore include genuinely advanced angles (internals, performance, concurrency/consistency trade-offs, failure modes, scale), not just the happy path.
- **Foundational early lessons** (e.g. "environment", "SELECT basics", "filesystem") still exist for completeness, but must carry an **"Advanced corner"** — deeper material that an experienced reader will still learn from (e.g. how `venv` works internally, `uv`/Poetry trade-offs, MVCC, `EXPLAIN` internals, page cache). Don't let any lesson read as purely introductory.
- **Tone & prose:** write **rich, professional, senior-engineer prose** — precise, opinionated where warranted, with real context and trade-offs. Quality and density over length, but never thin. This is a standing instruction for all future content.
- **Interview prep must scale with level:** include not just well-known questions but **mid/senior-level and "system design / trade-off" questions** with full, nuanced model answers.
- **Book & reference standard (REQUIRED):** every lesson's "Learn independently" section must cite **specific, named, authoritative books with the relevant chapters/sections**, plus primary docs and a few high-quality links — not vague "search online." Prefer the canonical works below and point to exact chapters when possible. Track the reader's request to always surface good books for any topic suggested.

### Canonical book shelf (default references to draw from)

**Python:** *Fluent Python* (Ramalho, 2nd ed) · *Effective Python* (Slatkin) · *Python Cookbook* (Beazley & Jones) · *CPython Internals* (Shaw) · *Architecture Patterns with Python* (Percival & Gregory) · *High Performance Python* (Gorelick & Ozsvald) · *Robust Python* (Viafore, for typing).

**SQL / data:** *Designing Data-Intensive Applications* (Kleppmann) — the data-engineering bible · *The Art of PostgreSQL* (Fontaine) · *PostgreSQL: Up and Running* (Obe & Hsu) · *SQL Performance Explained* (Winand; companion to use-the-index-luke.com) · *Database Internals* (Petrov) · *SQL for Smarties* (Celko, for advanced query/NULL logic).

**Linux / DevOps:** *The Linux Programming Interface* (Kerrisk) — the syscalls bible · *How Linux Works* (Ward) · *The Linux Command Line* (Shotts) · *UNIX and Linux System Administration Handbook* (Nemeth et al.) · *Systems Performance* (Brendan Gregg) · *BPF Performance Tools* (Gregg) · *Site Reliability Engineering* (Google, free online) · *The DevOps Handbook* (Kim et al.).

Secondary goal: the owner also wants to **master Git/GitHub** while building this repo (branching, PRs, releases, Actions). Treat git workflow as something to model well, and the `github/` track as first-class content.

## Repository structure

```
python/      15 lessons → generic Python cemented toward data eng / FastAPI / data science
sql/         15 lessons → PostgreSQL-first, confident toward data engineering + Kafka/streaming
linux/       15 lessons → toward DevOps / platform / Linux developer
combined/    cross-track exercises (2 or 3 technologies together)
capstones/   10 production-grade projects (built AFTER the 45 lessons)
github/      Git & GitHub mastery track
```

Each track folder has a `README.md` acting as its syllabus (the 15 topics + status). Lessons are files named `lesson-NN-kebab-topic.md` (e.g. `python/lesson-01-environment-and-execution-model.md`).

## Lesson authoring standard (MANDATORY)

Every lesson is a self-contained **~2-hour deep dive on ONE topic**, written for an experienced reader aiming at confident mid-level/senior depth (see "Audience, level & depth" above). Future instances MUST follow this exact section order so lessons stay consistent. Do not pad — depth over filler, but cover all sections.

1. **Lesson header** — track, lesson number, topic, estimated time, prerequisites, learning objectives.
2. **Why this matters** — where this sits in a career path.
3. **Theory** — thorough conceptual explanation, building from fundamentals.
4. **Official documentation quotes** — short *quoted* excerpts with source + URL (Python docs, PostgreSQL docs, man pages, POSIX, etc.). Quote accurately; do not invent quotes.
5. **Real-world examples** — runnable code / commands that actually work, with expected output.
6. **How This Is Used In Production** — REQUIRED section. Must cover: how startups use it, how enterprises use it, common architectures, scaling, monitoring, logging, security, CI/CD, common tools, and concrete examples referencing companies like Netflix, Uber, Amazon, Google, GitHub, Cloudflare, etc. (keep company claims general/plausible, not fabricated internal detail).
7. **Advanced corner** — REQUIRED. The deeper material that takes the reader past mid-level: internals, performance characteristics, concurrency/consistency trade-offs, failure modes, scale considerations, and "what most people get wrong at this level." Even foundational lessons must have substantive content here.
8. **Best practices** — do/don't, with rationale.
9. **Common mistakes & gotchas.**
10. **Where AI helps (and where it hurts)** — practical guidance on using AI tools for this topic.
11. **Learn independently** — REQUIRED to cite **specific named books with exact chapters/sections** (draw from the Canonical book shelf above), the primary official docs (with URLs), and a few high-quality links. No vague pointers.
12. **Interview preparation** — a mix of well-known questions AND mid/senior-level + system-design/trade-off questions, each WITH a full, nuanced model answer.
13. **Homework** — graded **Easy / Medium / Hard** tasks, with clear acceptance criteria. Solutions go in a `solutions/` subfolder or a collapsible section, never inline above the task. Tasks should stretch an experienced learner, not just drill syntax.
14. **Key takeaways** — bullet recap.

Combined exercises follow a lighter version of this but must state which lessons across tracks they assume.

## Capstone authoring standard

The 10 capstones live in `capstones/` and each combines multiple technologies to simulate real engineering. Every capstone MUST include, in order: Project overview · Business problem · Architecture diagram (**Mermaid**) · Requirements (functional + non-functional) · Folder structure · Step-by-step implementation guide · Production considerations · Security considerations · Performance improvements · Testing strategy · Common mistakes · Extensions for advanced learners · Interview discussion points · What a senior engineer would improve.

Example capstones (see `capstones/README.md`): Linux Monitoring Agent, Log Aggregation System, ETL Pipeline, PostgreSQL Backup Manager, File Synchronization Tool, Deployment Automation Tool, REST API with PostgreSQL, Server Health Dashboard, Database Migration Utility, Mini Platform Engineering Toolkit.

## Conventions

- **Markdown** is the medium. Use Mermaid for diagrams, fenced code blocks with language hints, and tables where they aid comparison.
- **PostgreSQL** is the default SQL dialect; note when something is standard SQL vs. Postgres-specific.
- **Python 3.12+** assumed; show modern idioms (type hints, `pathlib`, f-strings, `match`).
- Code and commands must be **runnable and correct** — verify logic before writing. Show expected output.
- Keep one topic per lesson, fully covered, rather than many topics shallowly.
- When adding a lesson, update that track's `README.md` status table.

## Working tips for future instances

- There is no build/test tooling yet; if example code grows enough to warrant running, prefer a per-lesson virtual environment and document the commands in the lesson itself.
- Generating all 45 lessons + 10 capstones is large; produce them in batches, keep the format identical, and keep the syllabus READMEs as the source of truth for what's done vs. pending.
