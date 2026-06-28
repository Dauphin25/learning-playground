# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A personal **6-month self-study curriculum** to master **Python, SQL, and Linux** — independently and combined — moving the owner from "Application Support with some prior experience" to a **strong junior+/mid-level engineer** ready for **data engineering, DevOps/platform engineering, and Python backend (FastAPI)** roles. It is a *content/learning* repository, not an application. The deliverables are lessons, exercises, and capstone project guides written in Markdown.

The full intent is captured in `description.md` — read it before authoring or restructuring content.

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

Every lesson is a self-contained **~2-hour deep dive on ONE topic**. Future instances MUST follow this exact section order so lessons stay consistent. Do not pad — depth over filler, but cover all sections.

1. **Lesson header** — track, lesson number, topic, estimated time, prerequisites, learning objectives.
2. **Why this matters** — where this sits in a career path.
3. **Theory** — thorough conceptual explanation, building from fundamentals.
4. **Official documentation quotes** — short *quoted* excerpts with source + URL (Python docs, PostgreSQL docs, man pages, POSIX, etc.). Quote accurately; do not invent quotes.
5. **Real-world examples** — runnable code / commands that actually work, with expected output.
6. **How This Is Used In Production** — REQUIRED section. Must cover: how startups use it, how enterprises use it, common architectures, scaling, monitoring, logging, security, CI/CD, common tools, and concrete examples referencing companies like Netflix, Uber, Amazon, Google, GitHub, Cloudflare, etc. (keep company claims general/plausible, not fabricated internal detail).
7. **Best practices** — do/don't, with rationale.
8. **Common mistakes & gotchas.**
9. **Where AI helps (and where it hurts)** — practical guidance on using AI tools for this topic.
10. **Learn independently** — curated pointers (docs, books, sites) for self-study.
11. **Interview preparation** — well-known interview questions WITH full model answers.
12. **Homework** — graded **Easy / Medium / Hard** tasks, with clear acceptance criteria. Solutions go in a `solutions/` subfolder or a collapsible section, never inline above the task.
13. **Key takeaways** — bullet recap.

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
