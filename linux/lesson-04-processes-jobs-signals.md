# Linux Lesson 04 — Processes, Jobs & Signals

| | |
|---|---|
| **Track** | Linux |
| **Lesson** | 04 of 15 |
| **Topic** | The process lifecycle, process state, job control, signals, and `/proc` |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–03 (shell, permissions, text processing) |

### Learning objectives
1. Explain the process lifecycle — `fork`, `exec`, `wait`, `exit` — and why Unix splits creation from execution.
2. Read process state (`R`, `S`, `D`, `Z`, `T`) and know what to do about each, especially `D` and `Z`.
3. Use `ps`, `top`/`htop`, `pgrep`/`pkill`, `lsof`, and `/proc` to answer real questions about a running system.
4. Use job control (`&`, `Ctrl-Z`, `bg`, `fg`, `jobs`, `nohup`, `disown`) and explain process groups, sessions, and controlling terminals.
5. Send and handle signals correctly — and explain precisely why `SIGKILL` is the wrong first choice and why it sometimes doesn't work at all.
6. Explain orphans vs zombies, PID 1's role, and why this matters enormously in containers.

---

## 1. Why this matters

Processes are the unit of execution, and everything you will do in DevOps is about them: a service won't start, a container exits immediately, a deploy hangs, a box is at load 40 with idle CPUs, a graceful shutdown drops requests. Every one of those is a process/signal question.

For your target roles:
- **DevOps/platform:** `systemd` (Lesson 10) is a process supervisor — it's signals and process groups with a config language. Kubernetes' `terminationGracePeriodSeconds` is literally "how long between `SIGTERM` and `SIGKILL`." A container that ignores `SIGTERM` gets `SIGKILL`ed and drops in-flight requests on every single deploy, and the symptom is a mysterious 0.1% error rate that nobody can reproduce.
- **Backend:** Gunicorn/uvicorn workers, graceful reloads, zombie reaping, and "why is my app not receiving `SIGTERM` in Docker?" (answer: because it's not PID 1, or because it *is* PID 1 and shells don't forward signals) are daily concerns.
- **Interviews:** "what happens when you press Ctrl-C?", "difference between `SIGTERM` and `SIGKILL`", "what's a zombie process and how do you kill one?" are near-universal. The last one is a trick question, and knowing why is the point.

---

## 2. Theory

### 2.1 What a process is

A process is a running instance of a program plus its kernel-held context: PID, parent PID (PPID), UID/GID (real, effective, saved), address space, file descriptor table, current working directory, signal dispositions, environment, and resource limits.

```bash
$ ps -o pid,ppid,uid,cmd -p $$
    PID    PPID   UID CMD
   4821    4820  1000 -bash
```
`$$` is the shell's own PID. `$PPID` is its parent (the terminal emulator or sshd).

The kernel exposes all of it under **`/proc/<pid>/`** — a virtual filesystem where reading a "file" runs kernel code:
```bash
$ ls /proc/$$/
cmdline  cwd  environ  exe  fd/  limits  maps  stat  status  task/  ...

$ cat /proc/$$/cmdline | tr '\0' ' '     # NUL-separated, hence the tr
-bash
$ readlink /proc/$$/exe                  # the actual binary, even if deleted/renamed
/usr/bin/bash
$ readlink /proc/$$/cwd                  # current directory — works even if you can't cd there
/home/you
$ ls -l /proc/$$/fd/                     # every open file descriptor
lrwx------ 1 you you 64 Jul 16 10:00 0 -> /dev/pts/0
lrwx------ 1 you you 64 Jul 16 10:00 1 -> /dev/pts/0
lrwx------ 1 you you 64 Jul 16 10:00 2 -> /dev/pts/0
```
`/proc` is not a documentation curiosity — it's how `ps`, `top`, `lsof`, and every monitoring agent get their data. When a tool isn't installed on a locked-down box, `/proc` still works. **`readlink /proc/PID/exe` and `ls /proc/PID/fd` have solved more production mysteries than any other pair of commands** — they answer "what is this process actually running?" and "what does it have open?" when nothing else will.

### 2.2 The lifecycle: `fork` + `exec`

Unix creates processes in two steps, which surprises people coming from Windows' `CreateProcess`:

1. **`fork()`** — duplicates the calling process. The child gets a copy of everything (address space, FDs, environment). `fork()` returns **twice**: `0` in the child, the child's PID in the parent.
2. **`exec()`** — *replaces* the current process image with a new program. Same PID, same FDs (unless marked close-on-exec), brand new code and memory. On success, `exec` **never returns**.
3. **`wait()`/`waitpid()`** — the parent blocks (or polls) until the child terminates, and collects its **exit status**.
4. **`exit()`** — the child terminates; the kernel keeps a small record until the parent `wait`s.

```c
pid_t pid = fork();
if (pid == 0) {                       // child
    execlp("ls", "ls", "-l", NULL);   // becomes ls; never returns on success
    _exit(127);                       // only reached if exec FAILED
} else {                              // parent
    int status;
    waitpid(pid, &status, 0);
    printf("child exited with %d\n", WEXITSTATUS(status));
}
```

**Why split them?** Because between `fork` and `exec` the child is a copy of the parent *running your code*, so it can set up the environment for the new program: redirect stdout to a file, close FDs, change UID, `chdir`, set resource limits. That gap is what makes shell redirection, pipes, and privilege dropping possible with no special API. It is a genuinely elegant design and **exactly the answer interviewers want** for "why fork+exec instead of spawn?"

**`fork` is cheap** because of **copy-on-write**: the child shares the parent's physical pages, marked read-only; the kernel copies a page only when either side writes to it. So forking a 4 GB process doesn't copy 4 GB. (It does copy the page tables, which is why forking a huge process still isn't free — and it's why `fork()` in a multi-threaded program is a minefield: only the calling thread survives in the child, so a mutex held by another thread is locked forever. That's why `posix_spawn`/`vfork` exist, and why Python's `multiprocessing` on Linux switched its default away from `fork`.)

When you run `ls -l` in bash, bash forks, the child execs `ls`, and bash `wait`s. When you run `cd`, bash does **not** fork — it can't, because a child changing directory wouldn't affect the parent. That's precisely why `cd` must be a **builtin**, and it's a lovely interview answer.

### 2.3 Process states

```bash
$ ps -eo pid,stat,comm --sort=-pcpu | head -6
    PID STAT COMMAND
   1234 R    stress
   4821 Ss   bash
    892 Ssl  sshd
   1567 D    dd
   1890 Z    defunct
```

| Code | State | Meaning | What you do |
|---|---|---|---|
| `R` | Running/runnable | on a CPU or in the run queue | normal |
| `S` | Interruptible sleep | waiting for an event; **can** be woken by a signal | normal — most processes |
| `D` | **Uninterruptible sleep** | waiting on I/O in the kernel; **ignores all signals, including SIGKILL** | investigate the I/O layer |
| `T` | Stopped | `SIGSTOP`/`SIGTSTP`, or under a debugger | `fg`/`bg`/`SIGCONT` |
| `Z` | **Zombie** | terminated; waiting for its parent to reap it | fix/kill the **parent** |
| `I` | Idle kernel thread | (kernel threads only) | ignore |

Modifiers: `s` session leader, `l` multi-threaded, `+` foreground process group, `<` high priority, `N` low priority.

**`D` state is the important one.** A process in `D` cannot be killed — not even by `SIGKILL` — because it's inside a kernel call that isn't at a point where signals can be delivered. It's blocked on I/O: a hung NFS mount, a dying disk, an overloaded SAN. The process is stuck until the I/O completes or errors out.

This also explains a classic confusion: **Linux load average counts `D` as well as `R`.** So a box with a hung NFS server shows load 40 with 0% CPU utilisation, because 40 processes are sitting in `D`. Load average on Linux is "runnable **plus** uninterruptible," which is not what most people assume and not what it means on other Unixes. Being able to explain that is a strong signal.

```bash
$ cat /proc/1567/stack    # what kernel function is it stuck in? (needs root)
$ cat /proc/1567/wchan    # the wait channel — a one-word answer
```

### 2.4 Signals

A signal is an asynchronous notification. Each has a default action; a process can **catch** (install a handler), **ignore**, or **block** most of them.

```bash
$ kill -l
 1) SIGHUP   2) SIGINT   3) SIGQUIT   4) SIGILL   5) SIGTRAP
 6) SIGABRT  7) SIGBUS   8) SIGFPE    9) SIGKILL 10) SIGUSR1
11) SIGSEGV 12) SIGUSR2 13) SIGPIPE  14) SIGALRM 15) SIGTERM
17) SIGCHLD 18) SIGCONT 19) SIGSTOP  20) SIGTSTP ...
```

The ones that matter:

| Signal | # | Default | Catchable? | Sent by |
|---|---|---|---|---|
| `SIGHUP` | 1 | terminate | yes | terminal closed; **by convention, "reload config"** for daemons |
| `SIGINT` | 2 | terminate | yes | **Ctrl-C** |
| `SIGQUIT` | 3 | terminate + **core dump** | yes | **Ctrl-\\** |
| `SIGKILL` | **9** | terminate | **NO** | `kill -9` — the kernel does it, the process never sees it |
| `SIGSEGV` | 11 | terminate + core | yes | invalid memory access |
| `SIGPIPE` | 13 | terminate | yes | wrote to a pipe with no reader |
| `SIGTERM` | **15** | terminate | yes | **`kill` with no argument — the polite default** |
| `SIGCHLD` | 17 | **ignore** | yes | a child terminated — this is the reaping trigger |
| `SIGCONT` | 18 | continue | yes | `fg`/`bg` |
| `SIGSTOP` | **19** | stop | **NO** | `kill -STOP` |
| `SIGTSTP` | 20 | stop | yes | **Ctrl-Z** |

**`SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored.** That's deliberate: the kernel needs a guaranteed way to remove and pause a process. Every other signal is negotiable.

**The `SIGTERM` → `SIGKILL` discipline:**
```bash
kill 1234              # SIGTERM: "please shut down cleanly"
sleep 10
kill -9 1234           # SIGKILL: "you're gone" — only after giving it a chance
```
`SIGTERM` lets the process flush buffers, close database connections, finish in-flight requests, and remove its PID file. `SIGKILL` gives it none of that: **buffered writes are lost, temp files leak, locks are left held, and half-written state stays half-written.** Reaching for `kill -9` first is the mark of someone who doesn't know the difference — and it's also *useless* against a `D`-state process, which is usually the actual reason `kill` "isn't working." So `kill -9` is simultaneously the more destructive option and, in the most common case where people reach for it, not even effective. That framing is a great interview answer.

Sending signals:
```bash
kill -TERM 1234              # by name (preferred — self-documenting)
kill -15 1234                # by number
kill -0 1234                 # send NOTHING — just test "does it exist and can I signal it?"
kill -TERM -1234             # NOTE the minus: signal the entire PROCESS GROUP 1234
pkill -f 'python.*worker'    # by full command line regex
pgrep -a -f worker           # find first, ALWAYS, before pkill
killall nginx                # by exact name (careful: on some Unixes this kills everything!)
```
**`kill -0` is underrated** — it's how PID-file-based scripts check liveness without side effects. And **always `pgrep` before `pkill`**: `pkill -f python` on a box running your app, an agent, and a cron job kills all three. That's a real outage, and it happens.

### 2.5 Job control, process groups, sessions

Three nested concepts, and the reason `nohup` exists:

- A **process group** is a set of related processes (a pipeline is one group). Signals can target a whole group with a negative PID.
- A **session** is a set of process groups, with one **controlling terminal** and one **session leader** (your shell).
- The **foreground process group** receives terminal-generated signals (`SIGINT` from Ctrl-C, `SIGTSTP` from Ctrl-Z). Background groups don't — and if a background group tries to *read* from the terminal, it gets `SIGTTIN` and stops.

```bash
$ ps -eo pid,ppid,pgid,sid,tty,stat,comm | head -4
    PID    PPID    PGID     SID TT       STAT COMMAND
   4821    4820    4821    4821 pts/0    Ss   bash          # session leader (Ss)
   5102    4821    5102    4821 pts/0    R+   sleep         # foreground (+)
```

Job control in bash:
```bash
$ sleep 300 &            # start in the background
[1] 5201                 # job number 1, PID 5201
$ sleep 400
^Z                       # Ctrl-Z -> SIGTSTP -> stopped
[2]+  Stopped   sleep 400
$ jobs -l
[1]-  5201 Running   sleep 300 &
[2]+  5202 Stopped   sleep 400
$ bg %2                  # SIGCONT, keep it in the background
$ fg %1                  # bring job 1 to the foreground
$ kill %2                # signal by JOB spec (note the %)
```

**The `SIGHUP` problem.** When your terminal closes (SSH drops), the kernel sends `SIGHUP` to the session's foreground process group, and bash forwards it to all its jobs. Default action: terminate. So your long-running job dies when your laptop sleeps.

Three fixes, and knowing the difference matters:
```bash
nohup ./long-job.sh &            # ignore SIGHUP; stdout/stderr -> nohup.out. Decide BEFORE starting.
./long-job.sh & disown -h %1     # remove from the shell's job table, or just mark it nohup-ish. AFTER starting.
setsid ./long-job.sh             # brand-new session, no controlling terminal at all. The cleanest.
tmux new -s work                 # a persistent session you can DETACH from and reattach later
```
**`tmux`/`screen` is the right answer for interactive work** — `nohup` gives you a detached process you can't get back to, whereas tmux lets you reattach, see the output, and interact. `nohup`/`setsid` are for scripts. `disown` is the "oh no, I forgot `nohup`" escape hatch. A senior habit: **start every long remote job inside tmux**, unconditionally.

### 2.6 Orphans, zombies, and PID 1

**Orphan:** a process whose parent died. The kernel **re-parents** it to PID 1 (or the nearest "subreaper"). Orphans are fine — they keep running, and PID 1 will reap them.

**Zombie (`Z`, "defunct"):** a process that has **already exited** but whose parent hasn't `wait()`ed to collect the exit status. The kernel keeps a tiny record (the PID and exit status) so the parent can still ask "how did my child do?"

```bash
$ ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
   1890   1888 Z    defunct
```

**Key facts, and this is a favourite interview question:**
- **You cannot kill a zombie.** It's already dead. `kill -9 <zombie_pid>` does nothing, because there's no process left to signal.
- A zombie consumes **no memory or CPU** — just a PID and a slot in the process table.
- **The bug is in the parent**, which isn't calling `wait()`. Fix or kill the *parent*; when the parent dies, the zombie is re-parented to PID 1, which reaps it immediately.
- Zombies matter only in bulk: thousands of them **exhaust the PID space** (`/proc/sys/kernel/pid_max`), and then *nothing* on the box can fork — you can't even run `ps` to diagnose it.

A correct parent reaps:
```bash
# in a shell script
./child &
wait $!                 # collect the exit status

# in C: install a SIGCHLD handler calling waitpid(-1, ..., WNOHANG) in a loop,
# or set SIG_IGN on SIGCHLD to have the kernel auto-reap.
```

**Why this matters enormously in containers.** In a container, **your app is PID 1**. PID 1 has two special properties:
1. It **inherits all orphans** and is therefore responsible for reaping them. If your app doesn't reap, zombies accumulate for the container's lifetime.
2. It gets **special signal treatment**: the kernel does **not** apply default actions for signals PID 1 hasn't installed a handler for. So `docker stop` sends `SIGTERM`, your app has no handler, and **nothing happens** — Docker waits 10 seconds, then `SIGKILL`s it. Every deploy hard-kills your app and drops in-flight requests.

The other half of the trap:
```dockerfile
CMD python app.py               # exec form -> python IS PID 1. Good (if it handles SIGTERM).
CMD ["sh", "-c", "python app.py"]  # sh is PID 1, python is its child.
                                   # sh does NOT forward SIGTERM. Your app never hears it.
```
Fixes: use the **exec form** (`CMD ["python", "app.py"]`), handle `SIGTERM` in your app, and/or use a minimal init (`tini`, `dumb-init`, or `docker run --init`) as PID 1 to forward signals and reap zombies. This is the highest-value practical takeaway in the lesson — it explains a whole category of "mysterious 0.1% error rate on deploy" that teams live with for years.

### 2.7 Priority and resources

```bash
nice -n 10 ./batch-job.sh        # start with lower priority (nice 10)
renice -n 5 -p 1234              # change a running process
renice -n -5 -p 1234             # NEGATIVE nice = HIGHER priority = requires root
ionice -c 3 -p 1234              # idle I/O class — for backups/rsync
```
Nice ranges −20 (highest priority) to +19 (lowest). Only root can go negative. `nice` affects **CPU** scheduling only — for a backup that's murdering your disk, you want **`ionice`**, and knowing the difference is a nice practical detail.

```bash
$ ulimit -a
core file size          (blocks, -c) 0
open files                      (-n) 1024        # <- the one that bites you
max user processes              (-u) 15721
$ ulimit -n 65535                # raise for this shell (soft limit, up to the hard limit)
$ cat /proc/1234/limits          # what a RUNNING process actually got
```
`ulimit -n` (max open FDs) is the classic production limit: every socket is an FD, so a server at 1024 connections starts throwing `EMFILE: too many open files`. The trap is that **limits are inherited at exec time** — raising `ulimit` in your shell does nothing for a service systemd already started; you must set `LimitNOFILE=` in the unit file. Checking `/proc/PID/limits` tells you what the process *actually* has, which is the only number that counts.

---

## 3. Official documentation quotes

> "`fork()` creates a new process by duplicating the calling process. The new process is referred to as the child process. The calling process is referred to as the parent process. The child process and the parent process run in separate memory spaces. At the time of `fork()` both memory spaces have the same content. Memory writes, file mappings (`mmap(2)`), and unmappings (`munmap(2)`) performed by one of the processes do not affect the other."
> — `man 2 fork`, [fork(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fork.2.html)

> "The `exec()` family of functions replaces the current process image with a new process image. … The `exec()` functions return only if an error has occurred."
> — `man 3 exec`, [exec(3) — Linux manual page](https://man7.org/linux/man-pages/man3/exec.3.html)

> "The signals `SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored."
> — `man 7 signal`, [signal(7) — Linux manual page](https://man7.org/linux/man-pages/man7/signal.7.html)

> "If a process terminates, and it has children in the zombie state, they are reparented to a subreaper process … A child that terminates, but has not been waited for becomes a 'zombie'. The kernel maintains a minimal set of information about the zombie process (PID, termination status, resource usage information) in order to allow the parent to later perform a wait to obtain information about the child. As long as a zombie is not removed from the system via a wait, it will consume a slot in the kernel process table, and if this table fills, it will not be possible to create further processes."
> — `man 2 wait`, [wait(2) — Linux manual page](https://man7.org/linux/man-pages/man2/wait.2.html)

> "The `/proc` filesystem is a pseudo-filesystem which provides an interface to kernel data structures. It is commonly mounted at `/proc`. Most of it is read-only, but some files allow kernel variables to be changed."
> — `man 5 proc`, [proc(5) — Linux manual page](https://man7.org/linux/man-pages/man5/proc.5.html)

> "PID 1 … has special significance: signals sent to it are not delivered unless the process has explicitly installed a signal handler for them. This is to prevent the accidental termination of the init process. The kernel will not apply default signal dispositions to PID 1."
> — paraphrasing `man 7 signal` and the init behaviour documented in [signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html) — *"the only signals that can be sent to process ID 1, the init process, are those for which init has explicitly installed signal handlers."*

---

## 4. Real-world examples

### 4.1 Watch fork/exec happen

```bash
$ strace -f -e trace=clone,execve,wait4 -o /tmp/trace.txt bash -c 'ls /tmp > /dev/null'
$ grep -E 'clone|execve|wait4' /tmp/trace.txt
execve("/usr/bin/bash", ["bash", "-c", "ls /tmp > /dev/null"], 0x7ffd...) = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|SIGCHLD, ...) = 5301
[pid  5301] execve("/usr/bin/ls", ["ls", "/tmp"], 0x55d...) = 0
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 5301
```
There it is, exactly as described: bash `clone`s (the modern `fork`), the child `execve`s `ls`, the parent `wait4`s and collects status 0. `strace -f` follows children — without `-f` you'd see only bash. **`strace` is the single best learning tool on Linux**, and this trace is worth reproducing yourself.

Now prove `cd` is different:
```bash
$ strace -f -e trace=clone,chdir -o /tmp/cd.txt bash -c 'cd /tmp'
$ grep -c clone /tmp/cd.txt
0                          # NO fork. cd is a builtin, by necessity.
```

### 4.2 Build a zombie, then reason about it

```bash
$ cat > /tmp/zombie.sh <<'EOF'
#!/bin/bash
# Parent that deliberately never wait()s.
sleep 1 &                 # child exits after 1s
echo "parent PID=$$  child PID=$!"
sleep 60                  # ...and the parent naps instead of reaping
EOF
$ chmod +x /tmp/zombie.sh
$ /tmp/zombie.sh &
parent PID=6001  child PID=6002

$ sleep 2 && ps -o pid,ppid,stat,comm -p 6002
    PID    PPID STAT COMMAND
   6002    6001 Z    sleep <defunct>          # a zombie

$ kill -9 6002                                # try to kill it
$ ps -o pid,stat,comm -p 6002
    PID STAT COMMAND
   6002 Z    sleep <defunct>                  # STILL THERE. It's already dead.

$ kill 6001                                   # kill the PARENT instead
$ ps -o pid -p 6002
    PID
                                              # gone — reparented to PID 1, which reaped it
```

That sequence *is* the interview answer, demonstrated. `kill -9` on a zombie is a no-op because there is no process to signal — only a bookkeeping record. Killing the parent works because the orphaned zombie gets re-parented to init, which reaps immediately.

Watch the PID exhaustion risk:
```bash
$ ps -eo stat | grep -c '^Z'          # zombie count — a legitimate thing to monitor
0
$ cat /proc/sys/kernel/pid_max
4194304
```

### 4.3 Graceful shutdown, done properly

```bash
$ cat > /tmp/graceful.sh <<'EOF'
#!/bin/bash
running=true
cleanup() {
    echo "[$(date +%T)] SIGTERM received — draining..."
    running=false
}
trap cleanup TERM INT

echo "[$(date +%T)] worker started, PID $$"
while $running; do
    echo "[$(date +%T)] ...working"
    sleep 1
done
echo "[$(date +%T)] flushed, closed, exiting cleanly"
exit 0
EOF
$ chmod +x /tmp/graceful.sh && /tmp/graceful.sh &
[1] 7001
[10:15:01] worker started, PID 7001
[10:15:01] ...working
[10:15:02] ...working
$ kill 7001
[10:15:03] SIGTERM received — draining...
[10:15:03] flushed, closed, exiting cleanly
[1]+  Done   /tmp/graceful.sh
```
Contrast with `kill -9 7001`: the process vanishes mid-line, no cleanup, no exit message. Now imagine those lines were "flush 10,000 buffered metrics" and "return 200 to the 12 in-flight HTTP requests."

**The shell-specific trap:** `trap` handlers don't run while bash is blocked in a foreground `sleep` — bash runs the handler only after the current command completes. For a shell script that must react instantly, background the sleep and `wait` on it:
```bash
sleep 30 & wait $!        # 'wait' IS interruptible by a trap. A plain 'sleep 30' is not.
```
This bites people writing entrypoint scripts, and it's exactly why entrypoint shell wrappers are a bad idea (§4.4).

### 4.4 The container PID 1 trap, demonstrated

```bash
$ cat > /tmp/app.py <<'EOF'
import signal, sys, time
def bye(sig, frame):
    print(f"got signal {sig}, shutting down cleanly", flush=True)
    sys.exit(0)
signal.signal(signal.SIGTERM, bye)
print("app running", flush=True)
while True: time.sleep(1)
EOF

# BAD: shell is PID 1 and does not forward SIGTERM
$ printf 'FROM python:3.12-slim\nCOPY app.py /\nCMD python /app.py\n' > /tmp/Dockerfile.bad

# GOOD: exec form — python IS PID 1
$ printf 'FROM python:3.12-slim\nCOPY app.py /\nCMD ["python", "/app.py"]\n' > /tmp/Dockerfile.good
```
```bash
$ docker build -f /tmp/Dockerfile.bad -t bad /tmp && docker run -d --name bad bad
$ time docker stop bad
bad
real    0m10.4s          # <-- TEN SECONDS. SIGTERM went to sh, which ignored it.
                         #     Docker waited, then SIGKILLed. Your app never knew.

$ docker build -f /tmp/Dockerfile.good -t good /tmp && docker run -d --name good good
$ time docker stop good
good
real    0m0.3s           # <-- instant, clean
$ docker logs good
app running
got signal 15, shutting down cleanly
```

**Ten seconds versus 0.3 seconds, and — far more importantly — every in-flight request dropped versus none.** This is arguably the most valuable 20 lines in the Linux track: it connects `fork`/`exec`, signals, PID 1's special status, and Docker's `CMD` forms into one observable failure that a huge number of production systems have right now. If your rolling deploys show a small error blip, this is the first thing to check.

### 4.5 Diagnosing a `D`-state process

```bash
$ uptime
 10:22:01 up 40 days, load average: 38.42, 35.10, 20.05
$ mpstat 1 1
%usr  %nice   %sys %iowait  %irq  %soft  %idle
0.12   0.00   0.31   94.20  0.00   0.02   5.35      # CPU idle, iowait pegged

$ ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /^D/'
   1567 D    nfs_wait_bit_killable   dd
   1568 D    nfs_wait_bit_killable   rsync
   ...

$ cat /proc/1567/stack           # needs root
[<0>] nfs_wait_bit_killable+0x2e/0x40
[<0>] out_of_line_wait_on_bit+0x8e/0xb0
```
Load 38 with an idle CPU and every offender in `D` on an NFS wait channel: **the NFS server is gone.** No amount of `kill -9` will help — the processes are inside a kernel call. Fix the storage, or `umount -f`/`umount -l` the mount. The diagnostic chain — high load + idle CPU + high iowait ⇒ look for `D` state ⇒ read `wchan` ⇒ blame the storage — is a complete, genuinely senior troubleshooting story, and it's the answer to "load is 40 but the CPUs are idle, what's happening?"

---

## 5. How This Is Used In Production

- **Startups:** `ps aux | grep`, `htop`, and `kill` are the daily loop; `tmux` keeps deploys alive over flaky wifi. The most common self-inflicted wound is `kill -9` as a reflex, which corrupts state (a `-9`'d Postgres or Redis loses unflushed writes and forces a recovery on restart), and the second is a Dockerfile using the shell form of `CMD`, silently hard-killing the app on every deploy.
- **Enterprises:** everything is a supervisor. **systemd** (Lesson 10) sends `SIGTERM`, waits `TimeoutStopSec`, then `SIGKILL`s — `KillMode=` controls whether the whole cgroup or just the main process is signalled. **Kubernetes** is the same contract at a higher level: `preStop` hook → `SIGTERM` → wait `terminationGracePeriodSeconds` (default **30s**) → `SIGKILL`. Getting graceful shutdown right is what makes zero-downtime rolling deploys actually zero-downtime; getting it wrong is a permanent error budget tax that presents as unexplainable 5xx blips during deploys.
- **Common architectures:** the **pre-fork worker model** (nginx, Gunicorn, Postgres, Apache) — one master forks N workers, uses `SIGCHLD` to notice deaths and re-fork, and does **zero-downtime reloads** by forking new workers with the new config, sending old workers `SIGQUIT`/`SIGWINCH` to finish in-flight requests and exit. That's `SIGHUP`-means-reload in action, and it's the single most common daemon design on Linux. Sidecars and init containers are the same fork/exec/wait relationships wearing YAML.
- **Scaling:** PID exhaustion is real — a fork bomb or a zombie leak fills `pid_max` and the box can no longer create *any* process, including your shell. cgroups' `pids.max` is the defence, and it's why every container should have one. FD exhaustion (`ulimit -n`) caps concurrent connections and is the classic "we hit 1024 users and everything broke." Thundering-herd signal delivery to thousands of processes is a real cost at scale.
- **Monitoring & logging:** `node_exporter` reads `/proc` — every process/CPU/load metric you see in Grafana came from there. Alert on: zombie count (a leak), `D`-state count (I/O trouble), load-vs-CPU divergence (the §4.5 story), FD usage against the limit, and OOM kills (`dmesg | grep -i oom` / `journalctl -k`). The **OOM killer** picks a victim by `oom_score` and sends an uncatchable kill — so an OOM'd process gets *no* graceful shutdown, which is why memory limits and graceful shutdown interact.
- **Security:** signals are an access-control boundary — you can only signal processes you own, unless you're root. `/proc/<pid>/environ` **leaks secrets** to anyone who can read it (which is why passing credentials via environment variables is weaker than a file or a secrets manager: any process running as the same user can read them, and so can a compromised sidecar). `hidepid=2` on `/proc` hides other users' processes. Never put a secret on a command line — `/proc/<pid>/cmdline` is world-readable and `ps aux` shows it to everyone on the box. Privilege dropping happens in the fork/exec gap: fork, `setuid` to an unprivileged user, then exec — which is precisely why the split exists.
- **CI/CD:** build timeouts are `SIGTERM` → `SIGKILL`. A CI job that leaves orphaned processes hangs the runner because the runner waits for all FDs on the pipe to close — the classic "job says done but never exits" is a background process still holding stdout. `timeout 300 ./flaky-thing` is the fix, and `timeout -k 10 300` sends `SIGKILL` 10 seconds after the `SIGTERM`.
- **Common tools:** `ps`, `top`/`htop`/`btop`, `pgrep`/`pkill`, `lsof`, `fuser`, `strace`/`ltrace`, `pidstat`, `pstree`, `systemd-cgls`, `tini`/`dumb-init`, `supervisord`, `tmux`/`screen`, `timeout`, `nice`/`ionice`, `perf` and `bpftrace` (Gregg's tooling — Lesson 12).
- **Real examples:** **Netflix**'s performance engineering (Brendan Gregg's work) is largely about reading `/proc` and tracing at scale — his USE method starts with exactly the load/state/wchan reasoning in §4.5. **Google**'s Borg (and therefore Kubernetes) formalised the `SIGTERM`-then-`SIGKILL` grace period as the universal container contract. **Cloudflare** and **nginx** deployments rely on the `SIGHUP`-reload/fork-new-workers pattern for config changes without dropping a single connection — that's why you can reload nginx under full production load and nothing notices.

---

## 6. Advanced corner

**`fork` is really `clone`.** Linux has one syscall — `clone()` — with flags controlling what's shared. `fork()` = `clone(SIGCHLD)` (share nothing). A **thread** = `clone(CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|...)` (share everything). So on Linux, **a thread is a process that shares more**, and both are `task_struct`s to the scheduler. That's why `ps -eLf` shows threads with LWP IDs, and why `/proc/<pid>/task/` lists them. Containers use the same call with `CLONE_NEWNS|CLONE_NEWPID|CLONE_NEWNET|...` — **a container is `clone()` with namespace flags**, which is the single best one-sentence explanation of containers and a direct bridge to Lesson 14.

**Copy-on-write and the fork+threads minefield.** COW makes `fork` cheap, but `fork()` in a multi-threaded process copies **only the calling thread**. Any mutex held by another thread at fork time is locked *forever* in the child, because the thread that would release it doesn't exist. Between `fork` and `exec` you may therefore only call **async-signal-safe** functions — `malloc` is not one of them, so a child that allocates before `exec` can deadlock. This is why POSIX has `pthread_atfork`, why `posix_spawn` exists, and why Python 3.14 made `forkserver`/`spawn` the default `multiprocessing` start method on Linux. It's also the root cause of a whole genre of "my app hangs randomly in production but never in dev" bugs.

**Signal handlers are brutally constrained.** A handler can interrupt your code *anywhere* — mid-`malloc`, mid-`printf`. So handlers may only call async-signal-safe functions (`man 7 signal-safety` lists them; `write` is in, `printf` is out). The idiomatic solution is the **self-pipe trick** — the handler does nothing but `write(pipefd, "x", 1)`, and the main event loop selects on that pipe, handling the signal in normal context. Modern Linux gives you `signalfd()`, which turns signals into readable FDs and fits an epoll loop natively; that's what most serious daemons use now. Standard signals also **do not queue** — if 100 `SIGCHLD`s arrive while one is blocked, you get **one**. That's why a `SIGCHLD` handler must loop `waitpid(-1, &st, WNOHANG)` until it returns 0, and forgetting the loop is a classic zombie-leak bug in hand-written supervisors. Real-time signals (`SIGRTMIN`+n) *do* queue.

**PID 1's kernel exemption, precisely.** The kernel refuses to apply *default* dispositions to PID 1 — signals with no installed handler are simply discarded (`man 7 signal`). This is to stop a stray `kill -TERM 1` from halting the machine. `SIGKILL` to PID 1 is also blocked; if PID 1 does die, the kernel panics. In a container's PID namespace, the namespace's PID 1 inherits this behaviour — which is exactly the §4.4 trap. It also inherits orphan-reaping duty *within the namespace*, which is why `tini` exists.

**PID reuse is a real race.** PIDs wrap at `pid_max` and get reused. A script that stores a PID, sleeps, then `kill`s it can signal an **entirely different process** that happened to be assigned that PID. This has caused real incidents (a cleanup script killing a database). Defences: `kill -0` plus verifying `/proc/<pid>/cmdline`, comparing process start time from `/proc/<pid>/stat` field 22, using cgroups (kill the whole cgroup — what systemd does), or **`pidfd_open()`** (Linux 5.3+), which gives you a stable file descriptor referring to a *specific* process, immune to reuse. `pidfd` is the modern correct answer and almost nobody knows it exists.

**Load average is not what you think.** Linux's is a 1/5/15-minute exponentially-damped moving average of tasks in `R` **plus `D`**. Other Unixes count only `R`. So on Linux, load includes disk-blocked tasks, which is why a hung NFS mount produces load 40 with idle CPUs — and why "load > core count means CPU saturation" is *wrong on Linux*. The correct interpretation needs `%iowait` and the `D`-state count alongside. Brendan Gregg's post on this is essential reading.

**`SIGPIPE` and why `head` doesn't hang.** `cat huge | head -1` terminates instantly: `head` exits after one line, closing the read end; `cat`'s next write gets `SIGPIPE`, whose default action is termination. That's the shell's flow control. In application code the trap is that this **kills your process** — so servers routinely `signal(SIGPIPE, SIG_IGN)` and check `write()`'s `EPIPE` return instead. Python ignores `SIGPIPE` by default and raises `BrokenPipeError`, which is why `python script.py | head` prints an ugly traceback where a C program dies silently.

**What most people get wrong at this level:**
- `kill -9` as the first move — it's both more destructive *and* useless against `D` state, which is when they usually reach for it.
- Trying to kill a zombie instead of fixing the parent.
- Shell-form `CMD` in Docker, and then living with a mystery deploy-time error rate for years.
- Believing load average measures CPU.
- Assuming a `trap` fires during a foreground `sleep` in bash.
- Not knowing threads and processes are the same kernel object with different `clone` flags.
- Storing PIDs and killing them later without guarding against reuse.

---

## 7. Best practices

- **`SIGTERM` first, always. `SIGKILL` only after a grace period.** Script it: `kill $pid; for i in $(seq 10); do kill -0 $pid 2>/dev/null || exit 0; sleep 1; done; kill -9 $pid`.
- **Handle `SIGTERM` in every long-running app.** Drain, flush, close, exit 0. This is what makes deploys invisible to users.
- **Use the exec form of `CMD`/`ENTRYPOINT`** in Docker, and add `--init`/`tini` if your app forks children or can't reap.
- **`pgrep` before `pkill`.** Every time. `pkill -f` matches more than you think.
- **Start long remote jobs in `tmux`**, not `nohup` — you can reattach and see what happened.
- Prefer **signal names** over numbers (`kill -TERM`), for readability and because numbers vary across architectures for the RT range.
- Use **`kill -0`** for liveness checks, and verify `/proc/PID/cmdline` before acting on a stored PID (or use cgroups/`pidfd`).
- Investigate `D` state at the **I/O layer** (`wchan`, `iostat`, `dmesg`) — signalling it is pointless.
- **Monitor zombie count and `D`-state count**, not just CPU. Alert on load-vs-CPU divergence.
- Use `nice` for CPU-bound background work and **`ionice -c 3`** for I/O-bound (backups, rsync) — the wrong one does nothing.
- Set `LimitNOFILE` in systemd units, not `ulimit` in a shell; verify with `/proc/PID/limits`.
- **Never put secrets in command lines or environment variables** on a shared box — `/proc/<pid>/cmdline` and `/proc/<pid>/environ` leak them.
- In shell scripts, background long sleeps and `wait` on them so traps can fire.

## 8. Common mistakes & gotchas

**`kill -9` as the reflex** — no cleanup, corrupted state, lost buffers. And it *won't work* on `D` state, which is usually why it seemed necessary.

**Trying to kill a zombie**
```bash
kill -9 <zombie_pid>       # no-op. It's already dead. Kill the PARENT.
```

**Docker shell-form `CMD`**
```dockerfile
CMD python app.py            # sh is PID 1; SIGTERM never reaches python; 10s then SIGKILL
CMD ["python", "app.py"]     # correct
```

**Assuming load average = CPU** — Linux counts `D` state too.

**`nohup` decided too late**
```bash
./job.sh              # oops, forgot nohup, and you need to log out
^Z ; bg ; disown -h %1   # the rescue sequence
```

**`pkill -f` overreach** — `pkill -f python` on a box with three Python things kills all three.

**PID reuse** — a stored PID may be someone else by the time you signal it.

**`trap` doesn't fire during a foreground `sleep`** in bash — the handler runs after the command completes. Use `sleep 30 & wait $!`.

**Forgetting the `waitpid` loop** in a `SIGCHLD` handler — signals don't queue, so N deaths can deliver one `SIGCHLD`, and you leak zombies.

**`ulimit` in a shell doesn't affect a running service** — limits are inherited at exec. Check `/proc/PID/limits`.

**Backgrounding without redirecting output** — a background job writing to the terminal interleaves with your prompt, and reading from it gets `SIGTTIN` and stops mysteriously.

**`SIGPIPE` killing your script** when its output goes to `head`/`less` and the reader exits early.

## 9. Where AI helps (and where it hurts)

- **Helps:** genuinely strong at explaining `strace` output and unfamiliar `wchan` values; writing the `SIGTERM`-then-`SIGKILL` drain loop correctly; producing `ps`/`awk` incantations for "show me all processes in D state sorted by..."; explaining the PID 1 trap once you describe the symptom.
- **Hurts:** AI suggests `kill -9` far too readily, because "how do I kill a stuck process" is answered with `-9` all over the internet. It will tell you to `kill -9` a zombie. It writes shell-form `CMD` in Dockerfiles by default, because that's the more common form in its training data — and this is a *real* bug it will hand you. It writes `SIGCHLD` handlers without the `waitpid` loop. And it rarely mentions that a `trap` won't fire during a foreground `sleep`. **Verify anything it tells you about signals against `man 7 signal`** — the man pages are precise and the internet isn't.

## 10. Learn independently

**Books (specific chapters):**
- *The Linux Programming Interface* (Kerrisk) — **the** reference, and the single most authoritative source for this lesson. **Ch. 24 "Process Creation"** (`fork`, COW, the fork-and-threads problem), **Ch. 25 "Process Termination"**, **Ch. 26 "Monitoring Child Processes"** (`wait`, zombies, `SIGCHLD` — read this one twice), **Ch. 27 "Program Execution"** (`exec`), **Ch. 20–22 "Signals"** (fundamentals, handlers, advanced features — the definitive treatment of async-signal-safety and the self-pipe trick), **Ch. 34 "Process Groups, Sessions, and Job Control"** (exactly §2.5), **Ch. 36 "Process Resources"** (`ulimit`, `rlimit`). Kerrisk wrote most of the Linux man pages; this book is those man pages with the *why* filled in. If you buy one Linux book, buy this one.
- *How Linux Works*, 3rd ed. (Ward) — **Ch. 1–2** (the process model, `/proc`) and **Ch. 6 "How User Space Starts"** — the best gentle-but-correct explanation of PID 1 and init, which is exactly the container trap's foundation.
- *The Linux Command Line* (Shotts) — **Ch. 10 "Processes"** — job control, `ps`, `kill` from the practical shell side. Free at [linuxcommand.org](https://linuxcommand.org/tlcl.php).
- *Systems Performance*, 2nd ed. (Gregg) — **Ch. 6 "CPUs"** (load average done properly, run queues) and **Ch. 2 "Methodologies"** (the USE method). Gregg's load-average material is the authoritative correction to the myth in §6.
- *UNIX and Linux System Administration Handbook*, 5th ed. (Nemeth et al.) — **Ch. 4 "Process Control"** — the operator's view, with the practical `nice`/`ionice`/troubleshooting material.

**Primary docs (man pages — read these, they're excellent):**
- [`man 7 signal`](https://man7.org/linux/man-pages/man7/signal.7.html) — **read this end to end.** The signal table, PID 1's exemption, queuing semantics.
- [`man 7 signal-safety`](https://man7.org/linux/man-pages/man7/signal-safety.7.html) — the async-signal-safe function list.
- [`man 2 fork`](https://man7.org/linux/man-pages/man2/fork.2.html), [`man 2 execve`](https://man7.org/linux/man-pages/man2/execve.2.html), [`man 2 wait`](https://man7.org/linux/man-pages/man2/wait.2.html) — the lifecycle, with the zombie paragraph.
- [`man 5 proc`](https://man7.org/linux/man-pages/man5/proc.5.html) — enormous and worth skimming; you'll return to it constantly.
- [`man 2 clone`](https://man7.org/linux/man-pages/man2/clone.2.html) — the flags that make threads and containers.
- [`man 2 pidfd_open`](https://man7.org/linux/man-pages/man2/pidfd_open.2.html) — the modern answer to PID reuse.
- [`man 1 bash`](https://man7.org/linux/man-pages/man1/bash.1.html) — the **JOB CONTROL** section.

**High-quality links:**
- [Brendan Gregg — "Linux Load Averages: Solving the Mystery"](https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html) — he traced the kernel history to find *why* `D` state is included. Required reading, and a great story.
- [Docker docs — "Specify signals"](https://docs.docker.com/reference/dockerfile/#stopsignal) and [tini](https://github.com/krallin/tini) / [dumb-init](https://github.com/Yelp/dumb-init) — the READMEs are the best short explanations of the PID 1 problem in existence.
- [Kubernetes — "Pod Lifecycle: termination"](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) — the `SIGTERM`/grace-period/`SIGKILL` contract, normatively.
- [`strace` tips](https://jvns.ca/blog/2015/04/14/strace-zine/) — Julia Evans' zines are the best on-ramp to `strace`, `/proc`, and signals. Genuinely delightful and genuinely accurate.

## 11. Interview preparation

**Q1. What happens when you press Ctrl-C?**
> The terminal driver recognises the `INTR` character and sends **`SIGINT`** to the **foreground process group** of the terminal's session — not to a single process, which is why Ctrl-C kills an entire pipeline. The default action is termination, but it's catchable, so a program can install a handler to clean up first (that's how `vim` ignores it and how Python raises `KeyboardInterrupt`). Background process groups don't receive it, which is exactly why `sleep 300 &` survives Ctrl-C. Related: Ctrl-Z sends `SIGTSTP` (catchable stop), Ctrl-\ sends `SIGQUIT` (terminate + core dump).

**Q2. `SIGTERM` vs `SIGKILL`?**
> `SIGTERM` (15) is the polite request — catchable, so the process can flush buffers, close connections, finish in-flight requests, remove its PID file, and exit cleanly. `SIGKILL` (9) is handled entirely by the kernel; the process never sees it and gets no chance to clean up, so you lose buffered writes and leave locks and temp files behind. The correct pattern is `SIGTERM`, wait a grace period, then `SIGKILL` — which is exactly what systemd (`TimeoutStopSec`) and Kubernetes (`terminationGracePeriodSeconds`) implement. Two things worth adding: `SIGKILL` and `SIGSTOP` are the only two signals that can't be caught, blocked, or ignored, by design — the kernel needs a guaranteed way to remove and pause a process. And `SIGKILL` **won't help against a `D`-state process**, which is usually the situation where people reach for it, so it's simultaneously the more destructive option and, in the common case, ineffective.

**Q3. What's a zombie process and how do you kill one?**
> A zombie has already **terminated**; the kernel retains a minimal record — PID, exit status, resource usage — so the parent can `wait()` and read it. **You cannot kill it, because it's already dead** — `kill -9` does nothing, there's no process to signal. The bug is in the **parent**, which isn't calling `wait()`/`waitpid()`. Fix the parent, or kill it: the zombie is then re-parented to PID 1, which reaps immediately. Individually harmless (no memory, no CPU), but thousands of them exhaust `pid_max` and then nothing on the box can fork at all — including the shell you'd use to diagnose it. The classic source is a hand-written supervisor with a `SIGCHLD` handler that calls `waitpid` once instead of looping — since standard signals don't queue, N simultaneous deaths deliver one `SIGCHLD` and you leak N−1 zombies.

**Q4. Zombie vs orphan?**
> An **orphan** is *alive* and its parent died; the kernel re-parents it to PID 1 (or the nearest subreaper), and it keeps running normally — this is how daemonisation traditionally worked. A **zombie** is *dead* and its parent is alive but hasn't reaped it. Opposite situations: orphan = live child, dead parent; zombie = dead child, negligent parent. Orphans are harmless by design; zombies are a parent bug.

**Q5. Explain `fork` and `exec`, and why they're separate.**
> `fork()` duplicates the calling process — it returns twice, `0` in the child and the child's PID in the parent — and it's cheap because of copy-on-write. `exec()` **replaces** the current process image with a new program, keeping the PID and file descriptors, and never returns on success. They're separate because the gap between them is where the child — running your code, in its own copy of the process — sets things up for the new program: redirect stdout to a file, close FDs, `chdir`, drop privileges with `setuid`, apply rlimits. **That gap is what makes shell redirection, pipes, and privilege dropping possible with no special API.** It's why `ls > out.txt` needs no cooperation from `ls`. The trade-off is that `fork` in a multi-threaded process is dangerous — only the calling thread survives, so any mutex held elsewhere is locked forever in the child — which is why `posix_spawn` exists and why Python moved `multiprocessing` off `fork` by default.

**Q6. What is `D` state and what do you do about it?**
> **Uninterruptible sleep** — the process is blocked inside a kernel call, typically waiting on I/O, at a point where signals can't be delivered. It ignores **everything, including `SIGKILL`**. You can't fix it from the process side; you fix the I/O — usually a hung NFS mount, a failing disk, or a saturated SAN. Diagnose with `/proc/PID/wchan` (the wait channel — one word telling you what it's blocked on) or `/proc/PID/stack` as root. It also explains a classic mystery: **Linux's load average counts `D` as well as `R`**, so a hung NFS server gives you load 40 with completely idle CPUs and high `%iowait`. That divergence — high load, idle CPU — is the signature, and it's the thing to look for before anyone reaches for `kill -9`.

**Q7. (System design) Your containerised app drops requests on every deploy. Diagnose.**
> Almost certainly the graceful shutdown contract. Kubernetes sends `SIGTERM`, waits `terminationGracePeriodSeconds` (default 30), then `SIGKILL`s. Three things break it, and I'd check them in this order. **(1) Signal never arrives:** the Dockerfile uses the shell form (`CMD python app.py`), so `/bin/sh` is PID 1 and doesn't forward `SIGTERM` — the app never hears it and gets hard-killed 30 seconds later. Fix: exec form, `CMD ["python", "app.py"]`. **(2) Signal arrives but isn't handled:** the app is PID 1, and the kernel doesn't apply *default* dispositions to PID 1 — so with no handler installed, `SIGTERM` is simply discarded. Fix: install a handler. **(3) Handled but the drain is wrong:** the app stops accepting new connections *and* closes existing ones immediately, instead of finishing in-flight work. Also — and this is the subtle one — pod termination and endpoint removal are **concurrent**, so the load balancer may still route to a pod that's already draining. The fix is a `preStop` hook with a few seconds' sleep, so the endpoint is removed *before* the app starts draining. I'd verify with `docker stop` locally and time it: instant means the signal path works, exactly-10-seconds means it's being ignored and Docker is timing out. That timing check is the fastest possible diagnostic.

**Q8. Load average is 40 on an 8-core box but CPU is idle. What's happening?**
> On Linux, load average counts tasks in `R` (runnable) **and `D`** (uninterruptible sleep) — unlike other Unixes, which count only runnable. So load 40 with idle CPUs means ~40 tasks are blocked in `D`, waiting on I/O that isn't completing. I'd confirm with `mpstat`/`vmstat` (expect high `%iowait`), find the offenders with `ps -eo pid,stat,wchan,comm | awk '$2 ~ /^D/'`, and read their wait channels — an NFS-related `wchan` points at a dead NFS server, a block-layer one at a failing disk. `dmesg` usually corroborates with I/O errors or NFS timeouts. The fix is at the storage layer; signalling the processes is pointless because `D` ignores `SIGKILL`. The general lesson is that **Linux's load average is a saturation metric across CPU and I/O, not a CPU metric** — which is why "load > cores means CPU-bound" is wrong here, and why you always read it alongside `%iowait` and the `D`-state count.

**Q9. (Trade-off) `nohup` vs `disown` vs `setsid` vs `tmux`?**
> `nohup` makes the process ignore `SIGHUP` and redirects output to `nohup.out` — but you must decide *before* starting. `disown` removes a job from bash's job table so bash won't forward `SIGHUP` on exit — the rescue when you forgot `nohup` (`^Z; bg; disown -h %1`). `setsid` starts the process in a brand-new session with **no controlling terminal**, so `SIGHUP` is structurally impossible — the cleanest, and what daemonisation traditionally did. **`tmux`/`screen`** is different in kind: it keeps a real terminal alive on the server that you can **detach from and reattach to**, so you keep interactivity and scrollback. For anything interactive or long-running over SSH, tmux is the right answer — the others give you a process you can't get back to, and "I can see it's still running but I can't see what it's doing" is a bad place to be during a migration. `nohup`/`setsid` are for scripts and `systemd-run` is arguably better than either on a modern box.

**Q10. What's the difference between a process and a thread on Linux?**
> Almost nothing at the kernel level — both are `task_struct`s to the scheduler, and both are created by **`clone()`** with different flags. `fork()` is `clone(SIGCHLD)` (share nothing); a thread is `clone(CLONE_VM|CLONE_FILES|CLONE_FS|CLONE_SIGHAND|CLONE_THREAD|...)` (share address space, FD table, signal handlers). So a thread is "a process that shares more." That's why `ps -eLf` shows threads with LWP IDs and why `/proc/PID/task/` lists them. The same syscall with namespace flags (`CLONE_NEWPID|CLONE_NEWNET|CLONE_NEWNS|…`) creates a **container** — which is genuinely the best one-sentence definition of a container: `clone()` with namespace flags plus a cgroup. Linux's uniformity here is unusual; most kernels have genuinely distinct abstractions.

**Q11. (Senior) Why can't a signal handler call `printf`?**
> Because a signal can interrupt the process **anywhere** — including in the middle of `printf` itself, or `malloc`. If the handler then calls `printf`, it re-enters a function that's mid-update on its internal locks and buffers, and you get corruption or a deadlock. Only **async-signal-safe** functions are legal in a handler (`man 7 signal-safety` lists them — `write` yes, `printf` no, `malloc` no). The idiomatic solutions: the **self-pipe trick**, where the handler does nothing but `write()` one byte to a pipe and the main event loop selects on it, handling the signal in normal context; or `signalfd()`, which turns signals into a readable FD that fits an epoll loop natively — what most modern daemons use. The general shape of the answer is "do the absolute minimum in the handler, set a `volatile sig_atomic_t` flag or write a byte, and handle it in the main loop."

**Q12. (Senior) How would you kill a specific process safely from a script six hours later?**
> Never by a stored PID alone — PIDs wrap at `pid_max` and get reused, so six hours later that PID may be an entirely different process, and scripts have killed databases this way. Escalating options: **(1)** store the PID and verify before signalling — `kill -0` plus check `/proc/PID/cmdline` matches and the start time in `/proc/PID/stat` field 22 matches what you recorded; the start time is the real guard, since it's unique per PID incarnation. **(2)** Use a **cgroup** — put the process in its own cgroup and kill the whole cgroup; membership is exact and reuse-proof. That's what systemd does, and `systemd-run --scope` gets you it for free. **(3)** Use **`pidfd_open()`** (Linux 5.3+), which returns a file descriptor referring to a *specific process instance*; `pidfd_send_signal()` on it can never hit the wrong process, even after reuse. That's the modern correct answer. **(4)** Don't manage it yourself — `systemd-run --on-active=6h` or a proper supervisor. In practice I'd use a systemd transient unit or a cgroup, because the whole class of problem disappears rather than being carefully guarded against.

## 12. Homework

> Work in `linux/solutions/lesson-04/`. Use a throwaway VM or container — several tasks are deliberately disruptive. Never run these on anything you care about.

**Easy**

1. **Read `/proc`.** For your own shell, print: the full command line, the real binary path, the CWD, all open FDs with their targets, the process state, and the number of threads — using **only** `/proc`, no `ps`. Then do the same for PID 1 and explain any differences.
   *Acceptance:* a script `inspect.sh <pid>` producing a labelled report; it must work with `ps` uninstalled (test with `PATH=/nonexistent bash inspect.sh $$`).

2. **The signal table.** Write a script that, for each of `SIGHUP`, `SIGINT`, `SIGQUIT`, `SIGKILL`, `SIGTERM`, `SIGSTOP`, `SIGTSTP`, `SIGUSR1`, prints the number, the default action, and whether it's catchable — sourced from `man 7 signal`, not memory. Then empirically prove `SIGKILL` and `SIGSTOP` can't be trapped by trying.
   *Acceptance:* the table, plus a script that installs traps for all eight and demonstrates which two don't fire.

3. **Job control drill.** In one session: start three background jobs, stop one with Ctrl-Z, resume it in the background, bring another to the foreground and stop it, kill one by job spec, and list the final state. Then show `PGID`/`SID`/`TTY` for all of them with `ps` and explain the `+`/`-`/`s` markers.
   *Acceptance:* an annotated transcript with the `ps` output and a one-line explanation of each state marker.

**Medium**

4. **Watch fork/exec.** Use `strace -f` to trace `bash -c 'ls | wc -l'`. Identify every `clone`, `execve`, `pipe`, `dup2`, and `wait4`, and write the sequence out as prose ("bash creates a pipe, forks, the child dup2s..."). Then trace `bash -c 'cd /tmp'` and show there's no fork.
   *Acceptance:* the annotated trace and the prose narration; you must correctly explain where the `dup2` calls come from and why there are two forks but one pipe.

5. **Graceful shutdown, three languages.** Write a worker in bash, Python, and one other language that: logs on start, works in a loop, catches `SIGTERM`, drains for 2 seconds, and exits 0. For each, demonstrate `kill` (clean) vs `kill -9` (abrupt) with timestamps. For the bash one, also demonstrate the `sleep`-blocks-traps problem and fix it with `sleep & wait $!`.
   *Acceptance:* three workers, six transcripts, and a `NOTES.md` explaining the bash trap/sleep interaction with evidence.

6. **Build and analyse a zombie.** Reproduce §4.2. Then: show the zombie's `/proc/<pid>/status` (what's still there?), prove `kill -9` does nothing, count zombies with a one-liner, kill the parent, and show the reaping. Finally, write a **correct** parent that reaps with `wait` and show no zombie appears.
   *Acceptance:* the full transcript, the zombie's `status` output with an explanation of which fields survive, and both parent versions.

7. **The container PID 1 trap.** Reproduce §4.4 fully: build both images, time `docker stop` on each, show the logs. Then add a third variant using `tini` (`--init`) with the shell form and show it fixes the *reaping* but **not** the signal forwarding through `sh -c`. Finally, add a fourth that forks a child and show zombie accumulation without `--init`.
   *Acceptance:* four containers, four `time docker stop` measurements, and a decision table: "use exec form when…, use `--init` when…, use both when…". **This is the most employable task in the lesson.**

**Hard**

8. **A supervisor in bash.** Write `supervise.sh` that starts N worker processes, restarts any that die (detect via `wait -n`), forwards `SIGTERM` to all children and waits for them to exit (with a `SIGKILL` fallback after 10 seconds), reaps properly so no zombies accumulate, and logs every event with timestamps. Test it by randomly killing workers with `-9` and by sending `SIGTERM` to the supervisor.
   *Acceptance:* survives 50 random worker kills with no zombies (`ps -eo stat | grep -c '^Z'` stays 0) and no orphans; `SIGTERM` to the supervisor produces a clean shutdown of all children within 11 seconds; a `-9`'d worker is restarted within 1 second.

9. **Simulate `D` state.** Using a network filesystem you control — an NFS server in a container, or `nbd`, or a `dm-delay` device — create a genuine `D`-state process. Show: the process ignoring `SIGKILL`, its `wchan`, the load average climbing while CPU stays idle, and `%iowait`. Then recover (restore the server, or `umount -l`).
   *Acceptance:* transcripts showing all four symptoms, an explanation of the load/CPU divergence, and a written runbook for "load is 40, CPUs are idle" that a colleague could follow at 3 a.m.

10. **PID reuse race, demonstrated.** Write a script that records a PID, waits, and kills it. Then force a PID wrap: lower `/proc/sys/kernel/pid_max` to something small on a throwaway VM, spawn processes in a loop until the PID is reused by a different program, and show your script killing the **wrong process**. Then fix it three ways: start-time verification from `/proc/PID/stat`, a cgroup, and `pidfd_open` (write the C or use Python 3.9+'s `os.pidfd_open`).
    *Acceptance:* documented evidence of the wrong process being killed, plus three working fixes with a note on which you'd use in production and why.

**Stretch (senior)**

11. **`signalfd` / self-pipe.** Write a program (C or Python) with an event loop that handles `SIGTERM`, `SIGINT`, `SIGCHLD`, and `SIGHUP` *without* doing work in the handler — use the self-pipe trick, then rewrite it with `signalfd`. Demonstrate the constraint that motivates this: write a handler that calls `printf` in a loop and show it can deadlock or corrupt output under signal storms (`while true; do kill -USR1 $pid; done`).
    *Acceptance:* both implementations working; evidence of the unsafe version misbehaving; and a written explanation of async-signal-safety citing `man 7 signal-safety`. Bonus: demonstrate that standard signals **don't queue** by sending 1,000 `SIGUSR1`s rapidly and counting how many the handler saw.

12. **Reproduce the fork-in-threads deadlock.** Write a multi-threaded program where one thread holds a `malloc` lock while another calls `fork()`, and the child then calls `malloc` before `exec` — and show it hanging. Explain the mechanism, then fix it three ways: `pthread_atfork`, `posix_spawn`, and calling only async-signal-safe functions between fork and exec. Then connect it to Python: show `multiprocessing` with the `fork` start method deadlocking in a threaded program, and fix it with `forkserver`/`spawn`.
    *Acceptance:* a reproducible hang (may need a few attempts — it's a race, so run it in a loop and report the hit rate), a stack trace from `gdb` showing where it's stuck, three C fixes, and the Python demonstration. Write up why Python changed its default start method; that's the payoff.

## 13. Key takeaways

- A process is `fork()` (duplicate, cheap via copy-on-write) + `exec()` (replace the image, never returns) + `wait()` (reap the status). **The gap between fork and exec is where redirection, pipes, and privilege dropping happen** — that's why Unix splits them.
- **`/proc/<pid>/`** is the interface to everything: `cmdline`, `exe`, `cwd`, `fd/`, `limits`, `status`, `wchan`. It's where `ps`, `top`, and every monitoring agent get their data — and it works when nothing else is installed.
- Process states: `R` runnable, `S` interruptible sleep, **`D` uninterruptible (ignores `SIGKILL`, counts toward load)**, `T` stopped, **`Z` zombie**.
- **`SIGTERM` then, after a grace period, `SIGKILL`.** `SIGKILL`/`SIGSTOP` are the only uncatchable signals. `kill -9` first is both destructive and, against `D` state, useless.
- **A zombie is already dead — you can't kill it.** Fix or kill the **parent**. Harmless individually; PID exhaustion in bulk.
- **In containers your app is PID 1**, which inherits orphan reaping *and* the kernel's rule that PID 1 gets no default signal actions. Shell-form `CMD` puts `sh` at PID 1 and it won't forward `SIGTERM` — that's a hard kill and dropped requests on **every deploy**. Exec form + a `SIGTERM` handler + `tini`/`--init`.
- Job control is process groups + sessions + a controlling terminal. `SIGHUP` on terminal close is why `nohup`/`disown`/`setsid` exist — but **use tmux** for interactive work.
- **Linux load average counts `D` as well as `R`** — high load with idle CPUs means blocked I/O, not CPU saturation.
- Threads, processes, and containers are all **`clone()` with different flags**. That single fact ties this lesson to Lesson 14.
- Signal handlers may only call async-signal-safe functions; standard signals **don't queue**. Use the self-pipe trick or `signalfd`, and always loop `waitpid(WNOHANG)` in a `SIGCHLD` handler.
