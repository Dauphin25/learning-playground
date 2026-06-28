# Linux Lesson 01 — Foundations, Filesystem Hierarchy & the Shell

| | |
|---|---|
| **Track** | Linux |
| **Lesson** | 01 of 15 |
| **Topic** | What Linux is, the filesystem hierarchy, and how to work effectively in the shell |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Access to a Linux shell (WSL2, a VM, a cloud VM, or Docker) |

### Learning objectives
1. Explain what "Linux" actually is (kernel vs distribution vs GNU userland).
2. Navigate the **Filesystem Hierarchy Standard** and explain what key directories hold.
3. Use the shell fluently: paths, navigation, globbing, tab completion, history.
4. Understand the command structure, `PATH`, and how the shell finds programs.
5. Read documentation on the box itself (`man`, `--help`, `tldr`).

---

## 1. Why this matters
Every server, container, and CI runner you'll touch as a backend/DevOps/data engineer is Linux. The shell is your primary interface to it — there's no GUI on a production box. Fluency here multiplies everything else: deploying apps, reading logs, debugging at 2 a.m. This lesson builds the map (filesystem) and the vehicle (shell) you'll use for the rest of the track.

---

## 2. Theory

### 2.1 Kernel vs distribution
- **Linux** is technically just the **kernel** — the program that manages CPU, memory, devices, and processes. Created by Linus Torvalds (1991).
- A **distribution (distro)** bundles the kernel + GNU userland tools + a package manager + defaults. Examples: **Ubuntu/Debian** (`apt`), **RHEL/Fedora/CentOS Stream** (`dnf`/`yum`), **Alpine** (`apk`, tiny — common in containers), **Arch** (`pacman`).
- The shell, `ls`, `grep`, etc. are mostly **GNU** tools, not the kernel — hence "GNU/Linux."

### 2.2 The shell
A **shell** is a program that reads commands and runs them. **Bash** (Bourne Again SHell) is the most common; **zsh** and **fish** are popular too; minimal containers ship **`sh`** (dash/busybox). The shell is also a programming language (Lessons 07–08).

When you type a command, the shell:
1. Splits the line into words (tokens), expanding globs/variables.
2. Finds the program (a builtin, or an executable on `$PATH`).
3. Forks a process, runs it, waits, and reports its **exit status** (`$?`, 0 = success).

### 2.3 The Filesystem Hierarchy Standard (FHS)
Everything is a file under a single root `/` (no drive letters). Key directories:

| Path | Purpose |
|------|---------|
| `/` | root of everything |
| `/bin`, `/usr/bin` | essential and user command binaries |
| `/sbin`, `/usr/sbin` | system administration binaries |
| `/etc` | system-wide **configuration** files (text) |
| `/home/<user>` | user home directories (`~`) |
| `/root` | the root user's home |
| `/var` | variable data: **logs** (`/var/log`), spools, caches |
| `/tmp` | temporary files (often wiped on reboot) |
| `/opt` | optional/third-party software |
| `/usr` | user programs, libraries, docs |
| `/lib`, `/usr/lib` | shared libraries |
| `/dev` | device files (`/dev/null`, `/dev/sda`) |
| `/proc`, `/sys` | virtual filesystems exposing kernel/process state |
| `/mnt`, `/media` | mount points for other filesystems |

> "Everything is a file" — devices, processes (`/proc`), and kernel knobs (`/sys`) are exposed as file-like objects you can read/write.

### 2.4 Paths
- **Absolute** path starts at root: `/var/log/syslog`.
- **Relative** path is from your current directory: `logs/app.log`, `../config`.
- Special: `.` (current dir), `..` (parent), `~` (home), `-` (previous dir with `cd -`).

### 2.5 Command structure
```
command  [options/flags]  [arguments]
ls       -la              /var/log
```
- Short flags: `-l`; combinable: `-la` = `-l -a`. Long flags: `--all`.
- `$PATH` is a colon-separated list of directories searched for commands: `echo $PATH`. `which ls` / `type ls` shows what runs.

### 2.6 Globbing (shell wildcards — different from regex!)
- `*` any characters, `?` one character, `[abc]` a set, `{a,b}` brace expansion.
```bash
ls *.log            # all .log files
ls report-?.txt     # report-1.txt, report-a.txt ...
cp file.{txt,bak}   # brace expansion → cp file.txt file.bak
```
The **shell** expands these *before* the command runs (regex in `grep` is a different system — Lesson 03).

---

## 3. Official documentation quotes

> "Linux is a clone of the operating system Unix, written from scratch by Linus Torvalds with assistance from a loosely-knit team of hackers across the Net. It aims towards POSIX and Single UNIX Specification compliance."
> — *The Linux Kernel Archives*, [kernel.org/category/about.html](https://www.kernel.org/category/about.html)

> "The Filesystem Hierarchy Standard (FHS) defines the directory structure and directory contents in Linux distributions."
> — *Filesystem Hierarchy Standard*, [refspecs.linuxfoundation.org](https://refspecs.linuxfoundation.org/fhs.shtml)

> "Bash is the GNU Project's shell — the Bourne Again SHell. This is an sh-compatible shell that incorporates useful features from the Korn shell (ksh) and the C shell (csh)."
> — *GNU Bash manual*, [gnu.org/software/bash](https://www.gnu.org/software/bash/)

---

## 4. Real-world examples

### 4.1 Orientation
```bash
whoami            # your username
hostname          # machine name
pwd               # print working directory
uname -a          # kernel + arch info
cat /etc/os-release   # which distro & version
```

### 4.2 Navigation & listing
```bash
cd /var/log
ls -lah           # long, all (incl. hidden), human-readable sizes
cd ~              # home
cd -              # previous directory
tree -L 2 /etc    # (if installed) directory tree, 2 levels
```

### 4.3 Inspect files quickly
```bash
cat /etc/os-release        # whole file
less /var/log/syslog       # paged view (q to quit, / to search)
head -n 20 file.log        # first 20 lines
tail -n 50 -f app.log      # last 50 lines, then follow new ones (great for live logs)
wc -l access.log           # count lines
file /bin/ls               # what kind of file is it
stat report.txt            # detailed metadata
```

### 4.4 Find the docs on the box
```bash
man ls                 # full manual (q to quit)
ls --help              # quick usage
type cd                # 'cd is a shell builtin'
which python3          # path to the binary
apropos network        # search man pages by keyword
```

---

## 6. How This Is Used In Production
- **Startups:** the entire app runs on Linux VMs or containers; engineers `ssh` in, `cd /var/log`, `tail -f` the app log, and `less` config in `/etc`. Knowing the FHS means you find things fast under pressure.
- **Enterprises:** standardized "golden images" (hardened Ubuntu/RHEL) across thousands of hosts; config lives in `/etc` managed by Ansible/Puppet; logs ship from `/var/log` to centralized logging. Shell fluency is assumed for every infra role.
- **Common architectures:** containers use *minimal* distros (Alpine, distroless) — same FHS, fewer tools, so you learn to do more with `sh` and what's available. `/proc` and `/sys` underpin metrics agents.
- **Scaling:** you rarely touch one box manually at scale — you script actions across fleets (Lessons 07–08, 15) — but every script is built from these primitives. `/proc` and `/sys` are how monitoring agents read CPU/mem/IO cheaply.
- **Monitoring/Logging:** `/var/log` + journald are the source; agents (Fluent Bit, Vector, the node exporter) read files and `/proc` and ship metrics/logs to Prometheus/Grafana/ELK.
- **Security:** least privilege (don't live as root), correct file locations and permissions (Lesson 02), and immutable/minimal images reduce attack surface.
- **Real examples:** Practically the entire cloud runs on Linux — **AWS, Google Cloud, Azure** hypervisors and the vast majority of their VMs; **Netflix** runs its streaming fleet on Linux (Ubuntu-based AMIs) on EC2; **Android** and most of the world's servers are Linux. The shell skills here are the literal day-job interface for SREs at all of them.

---

## 7. Best practices
- Use **tab completion** constantly (paths, commands) — fewer typos, faster.
- Use `less` for big files, `tail -f` for live logs; don't `cat` a 2 GB file.
- Prefer absolute paths in scripts/cron; relative paths interactively.
- Learn `man`/`--help` reflexes before searching the web — the answer is on the box.
- Keep `$HOME` tidy; put work in project dirs; never dump into `/`.
- Quote paths with spaces: `cd "My Folder"`.

## 8. Common mistakes & gotchas
- Confusing **shell globbing** (`*`) with **regex** (`.*`) — different systems (Lesson 03).
- Assuming `python`/`tools` exist in a minimal container — they may not.
- Editing config in `/etc` without a backup; one typo breaks a service.
- Running everything as `root` (or `sudo`) out of habit — dangerous.
- Forgetting that `cd` is a *builtin* (it changes the shell's own dir; a subprocess can't `cd` for you).
- Relative paths in cron jobs (cron's working dir/`$PATH` differ from your shell) — use absolute paths.

## 9. Where AI helps (and where it hurts)
- **Helps:** recalling a flag you forgot, explaining a `man` page in plain words, building a one-liner from a description, decoding `/etc/os-release` or `uname` output, suggesting which `/var/log` file to check.
- **Hurts:** AI may give commands for the wrong distro/package manager, suggest **destructive** commands (`rm -rf`, redirecting over files) without warning, or assume tools are installed. Read any deletion/overwrite command before running, and never paste a command you don't understand into a production shell.

## 10. Learn independently
- [The Linux Command Line](https://linuxcommand.org/tlcl.php) (William Shotts) — free, excellent.
- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/).
- [Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/fhs.shtml).
- `tldr` pages (`tldr ls`) for example-first help; [explainshell.com](https://explainshell.com/) to dissect commands.
- Practice in a throwaway environment: a cloud VM, WSL2, or `docker run -it ubuntu bash`.

## 11. Interview preparation

**Q1. Is Linux an operating system?**
> Strictly, Linux is the *kernel*. A usable OS is a *distribution*: the kernel plus GNU userland tools, libraries, a package manager, and defaults (Ubuntu, RHEL, Alpine). Hence "GNU/Linux."

**Q2. What's in `/etc`, `/var`, `/tmp`, `/proc`?**
> `/etc` = system configuration (text files). `/var` = variable data, notably logs in `/var/log`. `/tmp` = temporary files, often cleared on reboot. `/proc` = a virtual filesystem exposing kernel and per-process state as files.

**Q3. How does the shell find a command?**
> It checks if it's a shell builtin or alias; otherwise it searches each directory in `$PATH` in order and runs the first matching executable. `type cmd` / `which cmd` reveal what it resolves to.

**Q4. Difference between globbing and regular expressions?**
> Globbing is filename expansion done by the *shell* (`*`, `?`, `[]`, `{}`) before the command runs. Regex is a pattern-matching language used by tools like `grep`/`sed` on text content, with different syntax (`.*`, `^`, `$`). They look similar but are unrelated systems.

**Q5. Why can't a script `cd` for the shell that called it (or vice versa)?**
> `cd` changes the working directory of the *current process*. A script runs in a child process; its `cd` affects only that child and is gone when it exits. To change your shell's dir from a script you must `source` it (run it in the current shell).

**Q6. Absolute vs relative path — and why prefer absolute in cron/scripts?**
> Absolute starts at `/` and is unambiguous everywhere; relative depends on the current directory. Cron and services run with a different working dir and minimal `$PATH`, so relative paths break — use absolute paths there.

**Q7. How do you watch a log file update in real time?**
> `tail -f /var/log/app.log` (or `tail -F` to handle rotation), optionally `tail -n 100 -f` to show recent context first.

## 12. Homework
> Keep a `linux/solutions/lesson-01/notes.md` with commands and observations.

**Easy**
1. Identify your environment: run `uname -a`, `cat /etc/os-release`, `whoami`, `pwd`, `echo $PATH`. Record what distro/kernel you're on and how many directories are in your `PATH`.

**Medium**
2. Filesystem tour: for each of `/etc`, `/var/log`, `/bin`, `/proc`, `/home`, list its contents (top level), pick one interesting file, and describe in one sentence what it is (use `file`, `head`, `stat`). For `/proc`, `cat /proc/cpuinfo` and `/proc/meminfo` and note what they reveal.

**Hard**
3. Build a "system report" by hand (no scripting yet): using only navigation and inspection commands, produce a `report.md` answering — which distro & kernel, how much RAM (`/proc/meminfo`), how many CPUs (`/proc/cpuinfo`), the 5 largest files under `/var/log` (hint: `ls -laSh`), and the last 20 lines of the main system log. Note which commands you used for each.

**Stretch:** Demonstrate the `cd`-in-subshell point: write a one-line `sh -c 'cd /tmp; pwd'` and show your shell's `pwd` is unchanged afterward. Explain why in your notes.

## 13. Key takeaways
- "Linux" = kernel; a *distribution* adds the userland and package manager.
- One filesystem tree from `/`; learn the FHS — config in `/etc`, logs in `/var/log`, kernel/process state in `/proc`/`/sys`.
- The shell expands globs and resolves commands via `$PATH`, then runs them and reports an exit status.
- Globbing (shell) ≠ regex (tools). `cd` is a builtin affecting only the current process.
- `man`/`--help`/`tldr` put the docs on the box; `less`/`tail -f` are your log tools.
