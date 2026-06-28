# Python Lesson 01 — Environment & the Execution Model

| | |
|---|---|
| **Track** | Python |
| **Lesson** | 01 of 15 |
| **Topic** | How Python runs your code, and how to manage interpreters, virtual environments, and projects |
| **Estimated time** | ~2 hours |
| **Prerequisites** | A terminal you can type into (Linux Lesson 01 helps); ability to install software |

### Learning objectives
By the end you can:
1. Explain what happens between `python app.py` and your program running (source → bytecode → PVM).
2. Distinguish the interpreter, the standard library, and third-party packages.
3. Create and use isolated virtual environments, and explain *why* they exist.
4. Manage dependencies reproducibly with `pip` and a requirements/lock file.
5. Lay out a small Python project the way professionals do.

---

## 1. Why this matters

Almost every Python bug that wastes a junior's afternoon is really an *environment* bug: "works on my machine," a missing package, the wrong Python version, a globally-installed library shadowing yours. Data engineering and FastAPI work happen on servers and in containers where you cannot rely on an IDE to hide this from you. Mastering the execution model and environment management first means every later lesson runs cleanly — and it's the foundation interviewers probe to tell a tinkerer from an engineer.

---

## 2. Theory

### 2.1 What "running Python" actually means

Python is **compiled to bytecode, then interpreted by a virtual machine**. The steps:

```
your_source.py
   │  (1) tokenize + parse  →  Abstract Syntax Tree (AST)
   ▼
   │  (2) compile           →  bytecode (.pyc, stored in __pycache__)
   ▼
   │  (3) execute           →  the Python Virtual Machine (PVM) runs the bytecode
   ▼
program output
```

- **CPython** is the reference implementation (written in C) — the `python` you almost certainly have. Others exist: **PyPy** (JIT, faster long-running code), **Jython**, **GraalPy**. When people say "Python" they usually mean CPython's behavior.
- Bytecode is cached in `__pycache__/module.cpython-312.pyc` so unchanged modules don't recompile. It is *not* machine code and not a security boundary — it's an optimization.
- You can see bytecode with the `dis` module:

```python
import dis
def add(a, b):
    return a + b
dis.dis(add)
```
```
  2           0 RESUMING               0
  3           2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 BINARY_OP                0 (+)
             10 RETURN_VALUE
```

### 2.2 The GIL (introduce now, deep-dive in Lesson 13)
CPython has a **Global Interpreter Lock**: only one thread executes Python bytecode at a time. This makes single-thread code simple and fast but means CPU-bound threads don't run in parallel (you use multiprocessing or async for that). Keep this fact in your pocket; we return to it.

### 2.3 Three layers of code
1. **The interpreter** — `python` itself and its builtins (`len`, `print`, `dict`).
2. **The standard library** — batteries included, ships with Python (`os`, `json`, `pathlib`, `datetime`). No install needed.
3. **Third-party packages** — from the **Python Package Index (PyPI)**, installed with `pip` (`requests`, `fastapi`, `pandas`).

### 2.4 Why virtual environments exist
There is one system Python, but every project wants *different* versions of third-party packages. Installing globally creates conflicts: project A needs `pandas==1.5`, project B needs `pandas==2.2`. A **virtual environment** is a self-contained directory with its own `site-packages`, so each project's dependencies are isolated. This is non-negotiable professional practice.

### 2.5 How `import` finds things
When you `import foo`, Python searches `sys.path` in order: the script's directory, then `PYTHONPATH` entries, then the environment's `site-packages`. The first match wins — which is why naming your file `random.py` and then `import random` breaks mysteriously (your file shadows the stdlib). We cover packaging fully in Lesson 09.

---

## 3. Official documentation quotes

> "Python is an interpreted, object-oriented, high-level programming language with dynamic semantics."
> — *python.org*, [What is Python? Executive Summary](https://www.python.org/doc/essays/blurb/)

> "The module used to create and manage virtual environments is called `venv`. `venv` will usually install the most recent version of Python that you have available."
> — *Python docs*, [`venv` — Creation of virtual environments](https://docs.python.org/3/library/venv.html)

> "A virtual environment is created on top of an existing Python installation, known as the virtual environment's 'base' Python, and may optionally be isolated from the packages in the base environment, so only those explicitly installed in the virtual environment are available."
> — *Python docs*, [`venv`](https://docs.python.org/3/library/venv.html)

> "pip is the package installer for Python. You can use it to install packages from the Python Package Index and other indexes."
> — *pip documentation*, [pip.pypa.io](https://pip.pypa.io/en/stable/)

---

## 4. Real-world examples

### 4.1 Check what you have
```bash
python --version        # or python3 --version on many Linux/macOS systems
python -c "import sys; print(sys.executable)"   # which interpreter is this?
python -c "import sys; print(sys.path)"         # where imports are searched
```

### 4.2 Create and use a virtual environment
```bash
# create an env in a folder named .venv
python -m venv .venv

# activate it
source .venv/bin/activate        # Linux/macOS
.venv\Scripts\Activate.ps1       # Windows PowerShell

# now 'python' and 'pip' point INSIDE .venv
python -c "import sys; print(sys.executable)"   # .../.venv/bin/python

pip install requests
python -c "import requests; print(requests.__version__)"

deactivate                       # leave the env
```

### 4.3 Reproducible dependencies
```bash
pip install requests fastapi
pip freeze > requirements.txt        # snapshot EXACT versions
# later, on another machine / in CI / in a container:
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

A `requirements.txt` looks like:
```
fastapi==0.115.0
requests==2.32.3
```

### 4.4 A minimal professional project layout
```
my-tool/
├── .venv/                 # local env, NOT committed
├── .gitignore            # ignores .venv/, __pycache__/, *.pyc
├── README.md
├── requirements.txt
├── pyproject.toml         # modern project metadata (Lesson 15)
├── src/
│   └── my_tool/
│       ├── __init__.py
│       └── main.py
└── tests/
    └── test_main.py
```
`.gitignore` essentials:
```
.venv/
__pycache__/
*.pyc
.env
```

---

## 6. How This Is Used In Production

**Startups** typically pin dependencies in `requirements.txt` (or use [Poetry](https://python-poetry.org/) / [uv](https://github.com/astral-sh/uv) for lockfiles) and ship the app in a **Docker image** that bakes in one exact Python version + one resolved set of packages. The container *is* the environment — "works on my machine" disappears because the machine ships with the code.

**Enterprises** add: a private package index (Artifactory, AWS CodeArtifact, GitHub Packages) so builds don't depend on public PyPI uptime; **lockfiles with hashes** (`pip install --require-hashes`) to prevent supply-chain tampering; and automated dependency scanning (Dependabot, Snyk) on every PR.

**Common architecture:** developer venv → committed lockfile → CI builds a Docker image → image scanned and pushed to a registry → deployed to Kubernetes/ECS. The same image runs in dev, staging, and prod.

- **Scaling:** stateless app containers scale horizontally; the *environment* is identical across replicas because it's the same image hash.
- **Monitoring/Logging:** apps log to stdout (12-factor); a collector (Fluent Bit, Vector) ships logs to Elasticsearch/Loki. Errors go to Sentry.
- **Security:** pinned + hashed deps, minimal base images (`python:3.12-slim` or distroless), no secrets baked in (injected at runtime).
- **CI/CD:** GitHub Actions / GitLab CI runs `pip install -r`, linters, and tests in a fresh env on every push — exactly reproducing what you do locally.
- **Common tools:** `venv`, `pip`, `pip-tools`, `uv`, Poetry, `pyenv` (manage multiple Python versions), Docker.
- **Real examples:** Instagram famously runs a massive Django/Python monolith on CPython and has publicly discussed Python performance work; **Dropbox** built much of its desktop client and backend in Python and created `mypy`; **Google** uses Python broadly for tooling and infra. All of them depend on *reproducible* environments at scale — the principle scales from your laptop to theirs.

---

## 7. Best practices
- **One virtual environment per project.** Name it `.venv` and git-ignore it.
- **Never `sudo pip install`** into the system Python — you can break your OS's own tools.
- **Pin versions** for apps (`pip freeze`); use looser ranges only for libraries you publish.
- **Commit the lockfile/requirements, not the env.** The env is regenerated; the manifest is the source of truth.
- Prefer `python -m pip` / `python -m venv` over bare `pip`/`venv` so you're certain *which* Python you're acting on.
- Keep a `README.md` with the exact setup commands so a teammate (or future you) is productive in 60 seconds.

## 8. Common mistakes & gotchas
- **Forgetting to activate** the venv, then "pip installs nothing" or installs globally. Check `which python` / `sys.executable`.
- **Naming a file after a stdlib module** (`random.py`, `email.py`, `queue.py`) — your file shadows the real one.
- **Committing `.venv/`** — bloats the repo and is machine-specific.
- **`pip install` without a venv** on a shared/system Python — version conflicts and permission errors.
- Assuming `python` means Python 3 — on some systems `python` is Python 2; use `python3` or check.

## 9. Where AI helps (and where it hurts)
- **Helps:** generating `.gitignore`, explaining a cryptic `ImportError`/`ModuleNotFoundError`, scaffolding `pyproject.toml`, decoding `dis` output, suggesting why `which python` differs from expectations.
- **Hurts:** AI will confidently invent package names or versions that don't exist (typosquatting risk — always verify on PyPI), and may suggest `sudo pip install` or global installs. It can't see *your* activated environment, so it can't truly diagnose "which python is running" — you must run the command and feed it the output.

## 10. Learn independently
- Python docs: [Setup and Usage](https://docs.python.org/3/using/index.html) and [`venv`](https://docs.python.org/3/library/venv.html).
- [pip user guide](https://pip.pypa.io/en/stable/user_guide/).
- *Real Python* — "Python Virtual Environments: A Primer."
- The [Python Packaging User Guide](https://packaging.python.org/) (skim now, revisit in Lesson 15).
- Tool to explore later: [`uv`](https://github.com/astral-sh/uv) — a very fast, modern env/dependency manager.

## 11. Interview preparation

**Q1. Is Python compiled or interpreted?**
> Both, in a sense. CPython compiles your source to *bytecode* (cached in `__pycache__`), then an interpreter — the Python Virtual Machine — executes that bytecode. So there's a compilation step, but to portable bytecode, not native machine code (unlike C). PyPy adds a JIT that compiles hot bytecode to machine code at runtime.

**Q2. What is a virtual environment and why use one?**
> An isolated directory with its own `site-packages` and interpreter symlink, so each project has independent dependencies. It prevents version conflicts between projects and avoids polluting the system Python. Created with `python -m venv .venv`.

**Q3. What's the difference between `requirements.txt` and what `pip freeze` produces?**
> `pip freeze` outputs the *exact* installed versions of everything in the environment (a snapshot). `requirements.txt` is just a file `pip install -r` reads; people often generate it via `pip freeze`, but you can also hand-write looser constraints (e.g. `requests>=2.30`). For reproducible apps you want pinned versions; lockfiles (Poetry/uv/pip-tools) add transitive pinning + hashes.

**Q4. What is the GIL?**
> The Global Interpreter Lock — a mutex in CPython that allows only one thread to execute Python bytecode at a time. It simplifies memory management but means CPU-bound multithreading doesn't achieve true parallelism; you use `multiprocessing` or async I/O instead. (Deep dive in Lesson 13.)

**Q5. You get `ModuleNotFoundError: No module named 'requests'` but you installed it. Why?**
> Almost always you installed into a different interpreter than the one running the script — e.g. installed globally but the venv is active, or vice versa. Check `python -c "import sys; print(sys.executable)"` and `pip -V` to confirm they point to the same place. Use `python -m pip install` to guarantee they match.

**Q6. What does `python -m` do and why prefer it?**
> It runs a module as a script using the *current* `python` interpreter. `python -m pip ...` guarantees you use the pip belonging to that exact Python, avoiding "wrong pip" confusion on systems with multiple Pythons.

## 12. Homework

> Do these before peeking at any solution. Put your work under `python/solutions/lesson-01/`.

**Easy**
1. Create a project folder `env-lab/`, make a `.venv`, activate it, install `requests`, and run a one-liner that prints `requests.__version__`. Then `pip freeze > requirements.txt`. *Acceptance:* `requirements.txt` contains a pinned `requests==...` line.

**Medium**
2. Write `inspect_env.py` that prints: the Python version (`sys.version`), the interpreter path (`sys.executable`), whether it's running inside a venv (hint: compare `sys.prefix` and `sys.base_prefix`), and the first three entries of `sys.path`. Run it both inside and outside the venv and note the differences. *Acceptance:* the venv-detection line correctly says True inside and False outside.

**Hard**
3. Reproducibility drill: in `env-lab/`, write `main.py` that imports `requests` and fetches `https://httpbin.org/get`, printing the status code. Commit `requirements.txt`, `main.py`, and a proper `.gitignore` (excluding `.venv/`). Then **delete `.venv` entirely**, recreate it from `requirements.txt` only, and confirm `main.py` still runs. Write a 5-line `README.md` with the exact setup commands. *Acceptance:* a clean clone + the documented commands runs `main.py` successfully with no manual `pip install` of individual packages.

**Stretch (sets up Lesson 13):** run `python -m dis` on a small function and explain three of the bytecode instructions in your own words.

## 13. Key takeaways
- Source → bytecode (`__pycache__`) → executed by the PVM; CPython is the default implementation.
- Three layers: interpreter + standard library + PyPI packages.
- Virtual environments isolate per-project dependencies — one `.venv` per project, git-ignored.
- `pip freeze`/lockfiles + a committed manifest = reproducible environments, which is the whole game in production (Docker bakes this in).
- Always know *which* Python is running: `sys.executable`, `python -m pip`.
