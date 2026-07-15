# Linux Lesson 05 — Users, Groups, `sudo` & PAM Basics

| | |
|---|---|
| **Track** | Linux |
| **Lesson** | 05 of 15 |
| **Topic** | Identity, account management, privilege escalation, and the authentication stack |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–04 (permissions from Lesson 02 and processes from Lesson 04 especially) |

### Learning objectives
1. Explain UIDs/GIDs, and the difference between **real**, **effective**, and **saved** IDs — and why setuid works.
2. Read and reason about `/etc/passwd`, `/etc/shadow`, `/etc/group`, and `/etc/gshadow`, including the hash format.
3. Manage accounts properly: `useradd` vs `adduser`, system vs regular users, service accounts, and account lifecycle.
4. Configure `sudo` correctly — and recognise the rules that quietly grant full root.
5. Explain PAM's stack model (`auth`/`account`/`password`/`session` × `required`/`requisite`/`sufficient`/`optional`) well enough to read a real config.
6. Apply least privilege in practice: service accounts, `nologin` shells, SSH keys, and why `sudo su -` defeats the point.

---

## 1. Why this matters

Identity is the foundation of every security control on the box. Permissions (Lesson 02) only mean something because there's a UID to compare against; process ownership (Lesson 04) determines who can signal whom; every audit trail is a username. Get identity wrong and nothing above it holds.

For your target roles:
- **DevOps/platform:** you will create service accounts, write `sudoers` rules for deploy automation, debug "permission denied" that isn't about file modes, and answer auditors asking "who can become root and how do we know what they did?" A single sloppy `sudoers` line — `deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl *` — is full root, and you should be able to see that at a glance.
- **Backend:** containers run as UID 0 by default, which is a finding in every serious security review. Understanding UID namespaces and why `USER app` in a Dockerfile matters is table stakes.
- **Interviews:** "what does setuid do?", "how does sudo differ from su?", "what's in `/etc/shadow` and why isn't it in `/etc/passwd`?" are standard. "Why is `sudo vim` dangerous?" is the question that separates people who've read a hardening guide from people who understand the model.

---

## 2. Theory

### 2.1 Identity is a number

The kernel knows **numbers**. Usernames are a userspace convenience resolved through a name service. `chown ann file` stores UID 1001; if you delete `ann` and create `bob` with the same UID, bob owns the file. This is exactly why a `tar` archive restored on another box shows nonsense ownership, and why UID collisions across systems are a real problem in NFS environments.

```bash
$ id
uid=1000(you) gid=1000(you) groups=1000(you),27(sudo),999(docker)
$ id -u; id -g; id -Gn
1000
1000
you sudo docker
```
Note `docker` in there. **Membership in the `docker` group is equivalent to root**, because you can `docker run -v /:/host -u 0 --privileged` and own the machine. It's not a security boundary; it's a root-equivalent grant with a friendly name. Every hardening review flags this and every dev team is annoyed by it.

### 2.2 Real, effective, and saved UIDs — the mechanism behind setuid

Every process carries **three** UIDs (and three GIDs):

| ID | Meaning |
|---|---|
| **Real (RUID)** | who *started* the process — used for signal permission checks and accounting |
| **Effective (EUID)** | who the kernel checks for **access decisions** — this is the one that matters |
| **Saved (SUID)** | a stash, so a privileged process can drop privileges *temporarily* and get them back |

```bash
$ ps -o pid,ruid,euid,suid,comm -p $$
    PID  RUID  EUID  SUID COMMAND
   4821  1000  1000  1000 bash
```

Normally all three match. **The setuid bit** breaks that: when you exec a binary with the setuid bit set, the kernel sets the **EUID to the file's owner** while the RUID stays yours.

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 Mar 23 2024 /usr/bin/passwd
#   ^ 's' where 'x' would be = setuid
```
So when *you* run `passwd`, it runs with EUID 0 — which it must, because it writes `/etc/shadow`, mode `640 root:shadow`. But its RUID is still 1000, which is how `passwd` knows **whose** password to change. That's the whole design: **RUID says who you are, EUID says what you can do.** It's an elegant answer to a genuinely hard problem, and it's the canonical setuid interview answer.

The saved UID enables temporary drops:
```c
seteuid(getuid());     // drop to the real user for the risky part
open(user_file, ...);  // now this can't touch root-only files
seteuid(saved_uid);    // regain root for the privileged part
```
Without SUID, a process that dropped privilege could never get it back — so it would have to do everything as root, which is the opposite of what you want.

Find setuid binaries — this should be a short, boring list:
```bash
$ find / -perm -4000 -type f -ls 2>/dev/null
/usr/bin/passwd  /usr/bin/sudo  /usr/bin/su  /usr/bin/mount  /usr/bin/newgrp  ...
```
**Every setuid-root binary is a potential root exploit** — it's attacker-controlled input running with EUID 0. Historically these have been the richest source of local privilege escalation on Linux (see `pkexec`/CVE-2021-4034 "PwnKit", `sudo`/CVE-2021-3156 "Baron Samedit"). Modern practice is to eliminate them with **capabilities** (`CAP_NET_BIND_SERVICE` instead of setuid-root `ping`) and to audit the list — an unexpected entry is a compromise indicator, and a `find / -perm -4000` diff against a baseline is a cheap, effective detection.

### 2.3 The account files

**`/etc/passwd`** — world-readable, 7 colon-separated fields:
```
you:x:1000:1000:Your Name,,,:/home/you:/bin/bash
│   │ │    │    │            │          └─ shell
│   │ │    │    │            └─ home directory
│   │ │    │    └─ GECOS (full name, contact — historically a comment field)
│   │ │    └─ primary GID
│   │ └─ UID
│   └─ password placeholder: 'x' = "look in /etc/shadow"
└─ username
```
The `x` is the whole point. Passwords used to live here — but this file must be world-readable so `ls -l` can map UIDs to names, so **every user could read every hash and crack them offline.** Splitting the hashes into `/etc/shadow` (mode `640 root:shadow`) fixed it. That's the answer to "why two files?" and it's a nice illustration of how a security boundary gets retrofitted.

**`/etc/shadow`** — root-readable only, 9 fields:
```
you:$y$j9T$F5Jx...$q3kP...:19900:0:99999:7:::
│   │                       │     │ │     │
│   │                       │     │ │     └─ warn days before expiry
│   │                       │     │ └─ max age (days) — force a change
│   │                       │     └─ min age (days) — stop instant re-changes
│   │                       └─ last change (days since 1970-01-01)
│   └─ the hash
└─ username
```

The hash format is `$id$salt$hash`:

| `$id$` | Algorithm | Verdict |
|---|---|---|
| `$1$` | MD5 | **broken** — never |
| `$5$` / `$6$` | SHA-256 / SHA-512 | acceptable; `$6$` is the long-time default |
| `$2b$` | bcrypt | good |
| `$y$` | **yescrypt** | **the modern default** (Debian 11+, Fedora, RHEL 9) |
| `!` or `*` prefix | **locked** | `!` = locked by admin; `*` = no password ever set |

The `$id$` prefix makes the format self-describing, which is how the system verifies old hashes while issuing new ones in a better algorithm. **yescrypt** and bcrypt are *memory-hard*, which is what makes GPU cracking expensive — SHA-512 is fast, and "fast" is a liability for a password hash. That distinction (fast hash = bad for passwords) is worth being able to state.

Note the difference between `!` (locked password — the account can still authenticate via SSH key!) and a `nologin` shell (can't get an interactive session but can still authenticate). **Locking a password does not lock an account.** `usermod -L` sets `!`; to actually disable, you also need `usermod -s /usr/sbin/nologin`, `usermod -e 1` (expire), and — critically — to remove `~/.ssh/authorized_keys`. Missing that last step is one of the most common real offboarding failures.

**`/etc/group`** and **`/etc/gshadow`**:
```
sudo:x:27:you,deploy
│    │ │  └─ SUPPLEMENTARY members
│    │ └─ GID
│    └─ password placeholder (in /etc/gshadow — group passwords are a vestige, ignore them)
└─ group name
```
**Your primary group is in `/etc/passwd`, not here.** So a user's full group list is "the primary GID from `passwd`" ∪ "every group in `/etc/group` listing them." That's why `id` and `groups` sometimes surprise people, and why grepping `/etc/group` for a username misses their primary group.

### 2.4 Managing accounts

```bash
# Low-level, scriptable, POSIX-ish — what you use in automation:
useradd -m -s /bin/bash -G sudo,docker -c "Ann Smith" ann
passwd ann
usermod -aG docker ann          # -a is MANDATORY: without it, -G REPLACES all groups
userdel -r ann                  # -r also removes the home directory and mail spool

# Interactive, friendly, Debian-family:
adduser ann                     # prompts for everything, sets sane defaults
```
**`usermod -G docker ann` without `-a` removes ann from every other group**, including `sudo`. People have locked themselves out of their own servers this way. **Always `-aG`.** It's the single most dangerous flag omission in Linux user management.

**Service accounts** — the pattern you'll write hundreds of times:
```bash
useradd --system \
        --no-create-home \
        --shell /usr/sbin/nologin \
        --home-dir /nonexistent \
        --comment "myapp service account" \
        myapp
```
`--system` allocates a UID below `UID_MIN` (typically <1000, see `/etc/login.defs`), which by convention means "not a human" — login managers hide them, and it signals intent to anyone reading. `nologin` prevents interactive sessions. `--no-create-home` because a daemon doesn't need one. **This is the correct way to run any service**, and the alternative — running it as root — is the finding on every audit.

`nologin` vs `/bin/false`: both refuse a shell, but `nologin` prints a polite message (customisable in `/etc/nologin.txt`) and exits 1. Neither prevents `ssh user@host command` from... actually, both do prevent it, because ssh uses the login shell to run the command. But **neither prevents SSH port forwarding or SFTP**, which surprises people — if you want to *really* restrict, you need `Match User` blocks in `sshd_config`.

Defaults live in `/etc/login.defs` (UID ranges, password aging, hash algorithm) and `/etc/skel/` (files copied into every new home directory). Worth knowing both exist; `/etc/skel` is where you'd put a company-standard `.bashrc`.

### 2.5 `su` vs `sudo`

```bash
su                # become root; needs ROOT's password; keeps most of your environment
su -              # become root with a LOGIN shell — root's env, root's PATH, root's ~
su - ann          # become ann (root doesn't need a password)
sudo command      # run one command as root; needs YOUR password; logged
sudo -i           # a root login shell (like su -)
sudo -u ann cmd   # run as ann
sudo -l           # what am I allowed to do? — the first thing to run on a new box
```

The design difference matters:

| | `su` | `sudo` |
|---|---|---|
| Password | **root's** — shared secret | **your own** |
| Granularity | all or nothing | per-command, per-host, per-user |
| Audit trail | "someone became root" | "ann ran `/bin/systemctl restart nginx` at 10:42" |
| Revocation | change root's password **for everyone** | remove one line |
| Offboarding | rotate the shared secret | delete the user |

**`sudo` wins on every axis that matters operationally**, and the reason is that it eliminates the shared secret. With `su`, five admins share root's password; when one leaves, you rotate it and tell four people the new one — and the ex-employee still knows the old one worked. With `sudo`, you delete their account. The audit trail argument is equally decisive: `su` tells you *someone* became root; `sudo` tells you who ran what, when. Modern practice is to **disable root's password entirely** (`passwd -l root`, which Ubuntu does by default) so `su` to root is impossible and `sudo` is the only path.

The `-` matters more than people think:
```bash
$ su root -c 'echo $PATH'
/home/you/bin:/usr/local/bin:/usr/bin:/bin       # YOUR path — includes your writable dirs!
$ su - root -c 'echo $PATH'
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
Without `-`, root runs with **your** `PATH` — and if `~/bin` is in it and writable, an attacker who compromised your account has just put a trojan `ls` in root's path. `sudo` defends against this by default with `secure_path` in `/etc/sudoers` (§2.6), which is exactly why that setting exists.

### 2.6 `sudoers`

**Always edit with `visudo`.** It syntax-checks before saving. A broken `/etc/sudoers` means **nobody can use sudo**, and if root's password is disabled (the modern default), you are locked out of your own machine and need single-user mode or a rescue image.
```bash
visudo                          # the main file
visudo -f /etc/sudoers.d/deploy # a drop-in — PREFER THIS
visudo -c                       # check syntax without editing
```
Use `/etc/sudoers.d/` drop-ins: they're config-management friendly, they don't conflict on package upgrades, and one broken file is easier to identify. Note the gotcha: files in `sudoers.d` with a `.` or `~` in the name are **silently ignored** (it's a `run-parts` convention), so `deploy.conf` does nothing and you'll spend twenty minutes confused.

The rule grammar:
```
user    host = (runas_user:runas_group) TAG: command
ann     ALL  = (ALL:ALL)               ALL              # full root, with password
deploy  ALL  = (root)  NOPASSWD: /usr/bin/systemctl restart myapp
%sudo   ALL  = (ALL:ALL)               ALL              # % = a GROUP
%wheel  ALL  = (ALL)                   ALL              # RHEL-family convention
```

Aliases keep it readable at scale:
```
User_Alias  DEVS     = ann, bob, cleo
Cmnd_Alias  SERVICES = /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp
Host_Alias  WEB      = web01, web02

DEVS  WEB = (root) NOPASSWD: SERVICES
```

Defaults worth knowing:
```
Defaults    env_reset                  # scrub the environment — ON BY DEFAULT, leave it
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults    timestamp_timeout=15       # minutes before re-prompting; 0 = every time
Defaults    logfile="/var/log/sudo.log"
Defaults    log_input,log_output       # full session recording via sudoreplay
Defaults    requiretty                 # (RHEL) blocks sudo from non-tty — breaks automation
Defaults:deploy !requiretty            # per-user override
```
`env_reset` + `secure_path` is the defence against the `PATH` attack in §2.5 — **`sudo` builds a clean environment rather than trusting yours.** That's a deliberate, load-bearing design choice.

**Now the part that matters most: rules that look restricted but are full root.**

```bash
# Every one of these is equivalent to (ALL) ALL:
deploy ALL=(ALL) NOPASSWD: /usr/bin/vim          # :!sh   — shell escape
deploy ALL=(ALL) NOPASSWD: /usr/bin/less         # !sh    — pagers spawn shells
deploy ALL=(ALL) NOPASSWD: /usr/bin/find         # find . -exec sh \;
deploy ALL=(ALL) NOPASSWD: /usr/bin/awk          # awk 'BEGIN{system("sh")}'
deploy ALL=(ALL) NOPASSWD: /usr/bin/python3      # obviously
deploy ALL=(ALL) NOPASSWD: /bin/tar              # --to-command=sh, or --checkpoint-action=exec
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl    # systemctl edit / less pager / ExecStart=
deploy ALL=(ALL) NOPASSWD: /bin/cp               # cp /tmp/evil /etc/sudoers  (or /etc/shadow)
deploy ALL=(ALL) NOPASSWD: /usr/bin/git          # git -c core.pager='!sh' log
```

**Any binary that can spawn a subprocess, write an arbitrary file, or run a pager is root.** [GTFOBins](https://gtfobins.github.io/) catalogues hundreds. This is *the* practical `sudoers` lesson: reviewing a rule means asking "can this program be made to execute something else or write anywhere?", not "does this program sound dangerous?"

Wildcards are worse than they look:
```bash
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp*
# systemctl restart myapp../../../../etc/../bin/sh   -- the * matches far more than intended

deploy ALL=(ALL) NOPASSWD: /bin/chown deploy /var/www/*
# /var/www/../../etc/shadow  -- * doesn't stop at a slash in sudo's matching
```

**The right pattern** is a purpose-built, non-writable wrapper script that takes no arguments:
```bash
# /usr/local/sbin/restart-myapp  (root:root 755 — deploy must NOT be able to edit it)
#!/bin/bash
set -euo pipefail
exec /usr/bin/systemctl restart myapp.service
```
```
deploy ALL=(root) NOPASSWD: /usr/local/sbin/restart-myapp
```
No arguments, no wildcards, no shell escape, and the script isn't writable by the caller. **If the caller can write the script, the rule is root** — that's the check people forget.

Test everything with `sudo -l -U deploy`, which shows exactly what a user is allowed.

### 2.7 PAM — Pluggable Authentication Modules

PAM decouples *applications* from *authentication mechanisms*. `sshd`, `sudo`, `login`, and `su` don't know what a password is — they call PAM, and PAM consults a stack of modules. Add LDAP, or 2FA, or a fingerprint reader, and none of those programs change. That's the entire value proposition, and it's a good answer to "what is PAM for?"

Config lives in `/etc/pam.d/<service>`. Four **management groups**:

| Group | Question |
|---|---|
| `auth` | who are you? (verify credentials) |
| `account` | are you allowed in *right now*? (expired? time restrictions? locked?) |
| `password` | how do you change your credential? (complexity rules live here) |
| `session` | what happens around the session? (mount home, set limits, log, print motd) |

Four **control flags** (the part that takes a minute to click):

| Flag | On success | On failure |
|---|---|---|
| `required` | continue | **remember the failure**, keep going, fail at the end |
| `requisite` | continue | **fail immediately**, stop the stack |
| `sufficient` | **succeed now** (if nothing `required` already failed) | ignore, keep going |
| `optional` | ignored (unless it's the only module) | ignored |

`required` failing-late is deliberate: it stops an attacker learning *which* check failed from the timing or the error, which would leak whether a username exists. `requisite` fails fast when there's no point continuing (and no information leak to worry about). That "why does `required` keep going?" question has a real security answer and it's worth having ready.

A real `/etc/pam.d/sshd`, annotated:
```
auth       required     pam_env.so                 # set environment from /etc/environment
auth       [success=1 default=ignore] pam_unix.so nullok   # try local password
auth       requisite    pam_deny.so                # if we got here, no module succeeded -> deny
auth       required     pam_permit.so
auth       optional     pam_faildelay.so delay=2000000     # 2s delay on failure — anti-brute-force

account    required     pam_nologin.so             # /etc/nologin exists? nobody but root
account    required     pam_unix.so                # is the account expired/locked?

session    required     pam_limits.so              # apply /etc/security/limits.conf (ulimits!)
session    optional     pam_motd.so                # print the message of the day
session    optional     pam_mkhomedir.so           # create $HOME on first login (LDAP users)
```
The `[success=1 default=ignore]` form is the modern, more expressive syntax — it means "on success, skip the next 1 module." That's how the `pam_unix` → `pam_deny` pattern works: success jumps over the deny.

Modules you'll actually meet:
- **`pam_unix.so`** — the local `/etc/shadow` check.
- **`pam_pwquality.so`** / `pam_cracklib.so` — password complexity (`minlen`, `dcredit`, `retry`). Configured in `/etc/security/pwquality.conf`.
- **`pam_faillock.so`** — lock an account after N failures (replaces the older `pam_tally2`).
- **`pam_limits.so`** — applies `/etc/security/limits.conf`. **This is where `ulimit -n` for login sessions comes from** — a direct link to Lesson 04.
- **`pam_google_authenticator.so`** — TOTP 2FA.
- **`pam_sss.so`** / `pam_ldap.so` — enterprise directory integration.
- **`pam_wheel.so`** — restrict `su` to members of `wheel`.

**Editing PAM is genuinely dangerous.** A broken `/etc/pam.d/sshd` or `system-auth` can lock you out of the box entirely — this is not hypothetical, it's the classic way to brick a remote server. **Always keep a second root session open** while editing, and test the change in that new session before closing the working one. On a cloud VM, know how to get console access first.

---

## 3. Official documentation quotes

> "A set-user-ID program is one that has its set-user-ID permission bit set. When a set-user-ID program is run, the process's effective user ID is set to the user ID of the owner of the file. … The saved set-user-ID … allows a program to switch its effective user ID back and forth between the real user ID and the (initial) effective user ID."
> — `man 7 credentials`, [credentials(7) — Linux manual page](https://man7.org/linux/man-pages/man7/credentials.7.html)

> "`/etc/shadow` is a file which contains the password information for the system's accounts and optional aging information. This file must not be readable by regular users if password security is to be maintained."
> — `man 5 shadow`, [shadow(5) — Linux manual page](https://man7.org/linux/man-pages/man5/shadow.5.html)

> "The `sudo` (su 'do') command allows a system administrator to delegate authority to give certain users (or groups of users) the ability to run some (or all) commands as root or another user while providing an audit trail of the commands and their arguments."
> — `man 8 sudo`, [sudo(8) — sudo.ws](https://www.sudo.ws/docs/man/sudo.man/)

> "`env_reset` — If set, `sudo` will run the command in a minimal environment containing the `TERM`, `PATH`, `HOME`, `MAIL`, `SHELL`, `LOGNAME`, `USER` and `SUDO_*` variables. Any variables in the caller's environment or in the file specified by the `env_file` option that match the `env_keep` and `env_check` lists are then added, followed by any variables present in the file specified by the `restricted_env_file` option. … This flag is *on* by default."
> — `man 5 sudoers`, [sudoers(5) — sudo.ws](https://www.sudo.ws/docs/man/sudoers.man/)

> "It is not possible to run commands as root via `sudo` if the `sudoers` file is not readable … The `visudo` command should always be used to edit the `sudoers` file. It locks the file against multiple simultaneous edits, performs basic sanity checks, and checks for parse errors."
> — `man 8 visudo`, [visudo(8) — sudo.ws](https://www.sudo.ws/docs/man/visudo.man/)

> "`required` — failure of such a PAM will ultimately lead to the PAM-API returning failure but only after the remaining stacked modules (for this service and type) have been invoked. … `requisite` — like `required`, however, in the case that such a module returns a failure, control is directly returned to the application. … `sufficient` — success of such a module is enough to satisfy the authentication requirements of the stack of modules (if a prior required module has failed the success of this one is ignored)."
> — `man 5 pam.conf`, [pam.conf(5) — Linux-PAM](https://man7.org/linux/man-pages/man5/pam.conf.5.html)

---

## 4. Real-world examples

### 4.1 Watch setuid work

```bash
$ cat > /tmp/whoami-demo.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
int main(void) {
    printf("RUID=%d  EUID=%d\n", getuid(), geteuid());
    return 0;
}
EOF
$ gcc -o /tmp/whoami-demo /tmp/whoami-demo.c
$ /tmp/whoami-demo
RUID=1000  EUID=1000

$ sudo chown root:root /tmp/whoami-demo
$ sudo chmod u+s /tmp/whoami-demo
$ ls -l /tmp/whoami-demo
-rwsr-xr-x 1 root root 16696 Jul 16 10:30 /tmp/whoami-demo
$ /tmp/whoami-demo
RUID=1000  EUID=0                        # <-- run by you, EFFECTIVE root
```

That's the entire mechanism in six lines. RUID 1000 says "you started this"; EUID 0 says "the kernel will let this do root things." `passwd` uses exactly this: EUID 0 to write `/etc/shadow`, RUID 1000 to know whose entry to change.

Now the important part — **the danger**:
```bash
$ sudo cp /bin/bash /tmp/rootbash && sudo chmod u+s /tmp/rootbash
$ /tmp/rootbash -p                       # -p = don't drop privileges
# id
uid=1000(you) euid=0(root) groups=...
# cat /etc/shadow                        # full root
```
A setuid-root shell is instant, permanent root for anyone on the box. This is the single most common persistence backdoor an attacker leaves behind, which is why `find / -perm -4000` diffed against a known-good baseline is a genuinely effective detection. Clean it up: `sudo rm /tmp/rootbash /tmp/whoami-demo`.

(Note bash's `-p`: without it, bash *deliberately drops* EUID to RUID when they differ, precisely to defeat this. Many programs do this defensively. It's a good example of software knowing setuid is dangerous.)

### 4.2 Reading a real `/etc/shadow`

```bash
$ sudo getent shadow you | cut -c1-60
you:$y$j9T$SXo6iLBS/nQrDcvxOb.4a1$KzOAsn2mHmY1J...

$ sudo getent shadow | awk -F: '{print $1, substr($2,1,3)}' | head -8
root !          # locked — no password login possible (Ubuntu default)
daemon *        # never had a password
bin *
sys *
sync *
you $y$        # yescrypt
deploy $6$      # SHA-512 — older, should be rotated
backup !$6$     # LOCKED, but the hash is still there — unlock restores it
```
Reading this tells you the whole account posture at a glance: `root` locked (good — forces `sudo`), system accounts with `*` (never login-capable), `you` on yescrypt (modern), `deploy` on SHA-512 (works, but rotate it), `backup` locked *with the hash retained* (`!$6$` — `usermod -U` restores it instantly, which is why "locked" is not "disabled").

**Use `getent`, not `cat`.** `getent` goes through NSS, so it sees LDAP/SSSD/systemd-userdb users that aren't in the file at all. On any enterprise box, `cat /etc/passwd` shows you a fraction of the real users. That's a small habit that marks experience.

```bash
$ chage -l you
Last password change                                    : Jul 01, 2026
Password expires                                        : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
```

### 4.3 A `sudoers` rule reviewed properly

The request: "let CI restart our app."

```bash
# ATTEMPT 1 — what people write:
ci ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
```
Broken already. `systemctl` pages its output through `less` when stdout is a TTY, and `less` has `!sh`. Also `systemctl restart` isn't the only subcommand reachable if any wildcard creeps in later.

```bash
# ATTEMPT 2 — "I'll add a wildcard for flexibility":
ci ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp*
```
Much worse. `sudo`'s wildcards don't stop at `/`:
```bash
$ sudo systemctl restart myapp../../../../../../bin/sh    # matches the rule
```

```bash
# ATTEMPT 3 — the correct pattern:
$ sudo tee /usr/local/sbin/restart-myapp >/dev/null <<'EOF'
#!/bin/bash
set -euo pipefail
exec /usr/bin/systemctl --no-pager restart myapp.service
EOF
$ sudo chown root:root /usr/local/sbin/restart-myapp
$ sudo chmod 755 /usr/local/sbin/restart-myapp        # ci CANNOT write it — critical
$ sudo visudo -f /etc/sudoers.d/ci
```
```
ci ALL=(root) NOPASSWD: /usr/local/sbin/restart-myapp
```
```bash
$ sudo -l -U ci
User ci may run the following commands on this host:
    (root) NOPASSWD: /usr/local/sbin/restart-myapp
```
No arguments, no wildcards, `--no-pager` (no `less`, no `!sh`), and the script is root-owned and not writable by `ci`. Verify the last point explicitly — **if `ci` can edit the script, the rule is `(ALL) ALL`.** That's the check everyone forgets, and it's the one that matters.

Even better on a systemd box: skip `sudo` entirely with a polkit rule or `systemctl --user`, or grant the capability through systemd's own access control. But the wrapper pattern is universal and worth knowing.

### 4.4 A complete service-account setup

```bash
# 1. The account — no login, no home, system UID
$ sudo useradd --system --no-create-home --home-dir /nonexistent \
               --shell /usr/sbin/nologin --comment "myapp daemon" myapp
$ id myapp
uid=997(myapp) gid=997(myapp) groups=997(myapp)        # <1000 = system account

# 2. Directories — the app OWNS its data, READS its config
$ sudo mkdir -p /var/lib/myapp /var/log/myapp /etc/myapp
$ sudo chown myapp:myapp /var/lib/myapp /var/log/myapp
$ sudo chown root:myapp  /etc/myapp
$ sudo chmod 750 /var/lib/myapp /var/log/myapp
$ sudo chmod 750 /etc/myapp                            # app can READ config, not WRITE it
$ sudo chmod 640 /etc/myapp/secrets.env
$ sudo chown root:myapp /etc/myapp/secrets.env         # root writes, myapp reads

# 3. Prove it can't log in
$ sudo su - myapp
This account is currently not available.

# 4. Prove the isolation
$ sudo -u myapp cat /etc/myapp/secrets.env             # works — group read
$ sudo -u myapp sh -c 'echo x > /etc/myapp/secrets.env'
sh: /etc/myapp/secrets.env: Permission denied          # correct — can't tamper with its own config
$ sudo -u myapp cat /etc/shadow
cat: /etc/shadow: Permission denied                    # correct
```

The **config-owned-by-root, readable-by-the-app** pattern is the important detail. If the app owns its own config, an application compromise becomes config tampering becomes (via whatever the config can specify) privilege escalation. Root owns, the app reads. Same principle as the wrapper script in §4.3, and the same principle behind `/etc/sudoers` being root-owned.

### 4.5 Debugging PAM safely

```bash
# ALWAYS: keep this session open in another terminal before touching PAM.
$ sudo tail -f /var/log/auth.log        # Debian/Ubuntu
$ sudo journalctl -f -u sshd            # systemd
```
```bash
$ sudo pam-auth-update --list           # Debian: see the managed profiles
$ pamtester sshd you authenticate       # test WITHOUT risking a lockout — install pamtester
```
```
$ sudo grep 'sudo:' /var/log/auth.log | tail -3
Jul 16 10:42:01 web01 sudo: ann : TTY=pts/0 ; PWD=/home/ann ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
Jul 16 10:43:15 web01 sudo: bob : command not allowed ; TTY=pts/1 ; PWD=/tmp ; USER=root ; COMMAND=/bin/bash
Jul 16 10:43:20 web01 sudo:  bob : 3 incorrect password attempts ; TTY=pts/1 ; COMMAND=/bin/bash
```
That's the audit trail `su` can't give you — who, when, from where, what exactly, and whether it was permitted. Line two is a policy violation worth alerting on; line three is either a typo or an attack.

**`pamtester` is the tool that prevents lockouts.** It lets you verify a PAM stack change works before you trust your SSH session to it. Install it before you need it.

---

## 5. How This Is Used In Production

- **Startups:** one shared `ubuntu`/`ec2-user` account with a pile of SSH keys in `authorized_keys` — which works until you can't answer "who deployed at 3 a.m.?" or "did the person who left last month lose access?" The maturity step is individual accounts + `sudo` + centralised keys, and it costs an afternoon. The second is disabling password auth on SSH entirely and disabling root's password so `sudo` is the only path.
- **Enterprises:** identity is centralised — **SSSD** against Active Directory or LDAP, or increasingly short-lived certificates (Teleport, HashiCorp Boundary, AWS SSM Session Manager) with **no local accounts at all**. `sudoers` is managed by config management (Ansible/Puppet) or served from LDAP, never edited by hand. `sudo` logs ship to a SIEM. `log_input,log_output` gives full session recording via `sudoreplay` for privileged sessions, which auditors love. `pam_faillock` enforces lockout policy, `pam_pwquality` enforces complexity, and both exist mostly because a compliance framework says so.
- **Common architectures:** the **bastion/jump host** — one hardened entry point with all the auditing, and no direct access to anything else. **Zero-standing-privilege / just-in-time access** — nobody has `sudo` by default; you request elevation, it's approved, it expires. **Break-glass accounts** — a sealed root credential in a vault with alerting on use, for when the identity provider itself is down (a real scenario: if SSSD can't reach AD, nobody can log in, which is why local fallback matters).
- **Scaling:** UID/GID consistency across hosts is a genuine problem — with NFS, host A's UID 1001 is host B's UID 1001 regardless of names, so inconsistent local accounts mean users read each other's files. Central directory or explicit UID assignment in config management solves it. In containers, the same issue appears as "the file the container wrote is owned by 1000 on the host" — which is what **user namespaces** (`userns-remap`) fix by mapping container UID 0 to an unprivileged host UID.
- **Monitoring & logging:** alert on — `sudo` failures and "command not allowed" (policy violation or recon), any use of a break-glass account, new setuid binaries (diff `find / -perm -4000` against a baseline — cheap and effective), `/etc/passwd`/`/etc/shadow`/`/etc/sudoers` modification (`auditd` watch rules), new accounts with UID 0 (there should be exactly one), and successful logins from unexpected sources. `auditd` is the standard tool; `aide`/`tripwire` for file integrity.
- **Security:** this lesson *is* the security lesson. **Least privilege**: services run as dedicated non-root accounts with `nologin`; `sudo` is per-command; nobody logs in as root. **Container UID 0 is a real finding** — a container escape from UID 0 is root on the host unless user namespaces are on, so `USER app` in the Dockerfile plus a read-only root filesystem plus dropped capabilities is the baseline. **`docker` group = root** — treat it as such in your access model. **Capabilities over setuid**: `setcap cap_net_bind_service=+ep /usr/bin/myapp` lets it bind :80 without any root, replacing a whole class of setuid binaries. And the classic escalation paths to know: writable `sudoers.d` files, `sudo` rules on shell-escapable binaries (GTFOBins), writable wrapper scripts, `PATH` manipulation when `secure_path` is off, and stale `authorized_keys` after offboarding.
- **CI/CD:** deploy users get exactly one `NOPASSWD` wrapper script, never `(ALL) ALL`. The `requiretty` default on RHEL breaks automation, so you'll see `Defaults:deploy !requiretty` — understand why before copying it. Modern practice avoids SSH+sudo entirely: an agent (SSM, Ansible Pull) or a container orchestrator does the deploy, so there's no interactive privilege to escalate.
- **Common tools:** `useradd`/`usermod`/`userdel`, `getent`, `id`, `chage`, `visudo`, `sudo -l`, `pamtester`, `sssd`, `auditd`, `faillock`, `setcap`/`getcap`, `libpam-google-authenticator`, [GTFOBins](https://gtfobins.github.io/), `LinPEAS`/`linux-smart-enumeration` (attacker tools worth knowing defensively — they find exactly the misconfigurations in §2.6).
- **Real examples:** **Google**'s BeyondCorp eliminated the network perimeter in favour of per-request identity and device verification — the philosophical endpoint of "identity is the control." **Netflix**'s BLESS issues short-lived SSH certificates instead of managing `authorized_keys`, which solves offboarding structurally. **Facebook** and **Amazon** both published on eliminating standing root access in favour of just-in-time elevation. And CVE-2021-4034 (`pkexec`/PwnKit) and CVE-2021-3156 (`sudo`/Baron Samedit) are the two most instructive recent reminders that **every setuid binary is attack surface** — both gave instant local root on essentially every Linux box on earth, from bugs in the exact machinery this lesson describes.

---

## 6. Advanced corner

**Capabilities: setuid, decomposed.** Setuid is all-or-nothing — `ping` needed root just to open a raw socket, so a bug in `ping` was root. Linux split root's powers into ~40 **capabilities**:
```bash
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep                    # NOT setuid-root anymore
$ sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/myserver   # bind :80 as a normal user
$ getpcaps $$                                   # what does this process have?
```
Each has a **capability set** — permitted, effective, inheritable, bounding, ambient. `CAP_SYS_ADMIN` is the notorious one: it's so broad ("mount, namespaces, and ~30 unrelated things") that it's effectively root, which is why "just add `CAP_SYS_ADMIN`" in a Kubernetes securityContext is not the harmless-sounding fix it appears to be. **`CAP_DAC_OVERRIDE` bypasses all file permission checks**; `CAP_SETUID` lets you become anyone. Docker drops most capabilities by default and `--privileged` gives them all back, which is why `--privileged` is essentially `--i-am-root-on-the-host`.

**`sudo` reads its config as root — and that's a trust chain.** If any file in `/etc/sudoers.d/` is writable by a non-root user, that user is root. So is anyone who can write a wrapper script referenced in a rule. So is anyone in a group granted a rule, if group membership is manageable. **The trust chain for a `sudoers` rule includes every file the rule references and every file that could modify those files.** Auditing `sudoers` means auditing that whole closure, not reading one file. That framing is the senior insight here.

**PAM stacks are ordered, and order is security.** Putting `pam_permit.so` above a `required` check does nothing (`required` fails late), but putting `sufficient` above one **skips it entirely** — a `sufficient` that succeeds short-circuits the rest of the stack. So a single misordered `sufficient` line can bypass your 2FA module completely, and it will look fine because logins still work. This is why PAM edits go through config management with review, why `pam-auth-update` exists to manage the ordering for you on Debian, and why `pamtester` is worth installing before you need it.

**NSS vs PAM — different jobs, constantly confused.** **NSS** (`/etc/nsswitch.conf`) answers *"who is UID 1001?"* and *"does user ann exist?"* — it's identity **lookup**, used by `getent`, `id`, `ls -l`. **PAM** answers *"can this person prove they're ann?"* — it's **authentication**. A user can exist via NSS (LDAP) but fail PAM (wrong password), or authenticate via PAM but have no NSS entry (and then have no home, no shell, no UID — a genuinely confusing failure mode). When "the user exists but can't log in," check PAM; when "login works but `id` says unknown user," check NSS. Being able to make that split immediately saves an hour.

**Group membership is cached in the process, not looked up live.** Your supplementary groups are set at **login** (via `initgroups()`) and inherited by every child. So `usermod -aG docker ann` **does not affect ann's current session** — she must log out and back in, or use `newgrp docker` / `sg docker -c cmd`. This confuses everyone at least once ("I added myself to docker and it still says permission denied"). Related: `su - ann` re-runs `initgroups`, but `su ann` does too — while a running daemon never re-reads its groups, so adding a service account to a group requires a **service restart**, not just a config change.

**Passwords are hashed with memory-hard functions for a reason.** SHA-512 (`$6$`) is *fast* — a GPU does billions per second, so a cracker with a leaked shadow file works through common passwords quickly. **bcrypt** and **yescrypt** are deliberately memory-hard: they need a large working set, and GPUs have lots of cores but limited per-core memory bandwidth, so the parallelism advantage collapses. That's why yescrypt is now the default. The general principle — **a password hash should be slow and memory-hungry, unlike every other hash you'd ever want** — is worth internalising, because it explains Argon2, scrypt, and PBKDF2 too.

**`root` is UID 0, not the name.** The kernel checks the *number*. So `useradd -o -u 0 backdoor` creates a second full-root account named `backdoor` (`-o` allows a duplicate UID), and it will not appear in any "who is root?" check that greps for the name. `awk -F: '$3==0' /etc/passwd` is the real check, and finding more than one line is an incident.

**What most people get wrong at this level:**
- `usermod -G` without `-a`, nuking their own group memberships.
- Believing `sudoers` rules on specific binaries are restrictive (GTFOBins says otherwise).
- Thinking `usermod -L` disables an account — it locks the *password*; SSH keys still work.
- Confusing PAM (authn) with NSS (lookup) and debugging the wrong one.
- Not knowing group changes need a re-login (or a service restart).
- `cat /etc/passwd` instead of `getent passwd` on a box with LDAP.
- Treating `docker` group membership as less than root.
- Running containers as UID 0 because "it's isolated anyway."

---

## 7. Best practices

- **Always `usermod -aG`.** Never bare `-G`. It's the highest-consequence flag omission in this lesson.
- **Always `visudo`** (or `visudo -f` for drop-ins), and prefer `/etc/sudoers.d/` files — remembering that names with `.` or `~` are silently ignored.
- **Never grant `sudo` on a binary that can spawn a shell, write arbitrary files, or invoke a pager.** Check [GTFOBins](https://gtfobins.github.io/) before writing any rule. When in doubt, wrapper script.
- **Wrapper scripts take no arguments, use no wildcards, and are root-owned and not writable by the caller.** Verify the last part explicitly.
- **`sudo`, not `su`.** Disable root's password (`passwd -l root`). Individual accounts, no shared credentials.
- **`sudo -l -U user` is how you review**, not reading the file. It resolves aliases and shows the effective policy.
- **Service accounts:** `--system --no-create-home --shell /usr/sbin/nologin`. Config owned by **root**, readable by the service. Never let a service own its own config.
- **Offboarding is four steps**, and everyone forgets one: lock the password (`usermod -L`), set `nologin`, expire the account (`usermod -e 1`), and **remove `~/.ssh/authorized_keys`**. The last one is the one that matters.
- **Keep a second root session open** when editing PAM or `sudoers`. Test with `pamtester`/`visudo -c` before trusting it.
- **Audit setuid binaries** — baseline `find / -perm -4000` and diff it. Prefer **capabilities** (`setcap`) over setuid where you control the binary.
- **`getent`, not `cat`** — it sees LDAP/SSSD users that files don't.
- **Check for extra UID 0 accounts**: `awk -F: '$3==0' /etc/passwd` should return exactly one line.
- **Containers: `USER app`, non-root, drop capabilities, read-only rootfs.** Treat `docker` group membership as root in your access model.
- Log `sudo` centrally; alert on failures, "command not allowed," and break-glass use.

## 8. Common mistakes & gotchas

**`usermod -G` without `-a`**
```bash
usermod -G docker ann        # ann is now ONLY in docker. Not in sudo. Not in her own group.
usermod -aG docker ann       # correct
```

**Group changes don't apply to existing sessions** — log out/in, or `newgrp`. Daemons need a **restart**.

**`sudo` on an escapable binary is full root** — `vim`, `less`, `find`, `awk`, `tar`, `git`, `systemctl`, `cp`, `python`. All of them.

**`sudoers` wildcards don't stop at `/`**
```bash
deploy ALL=(ALL) /bin/chown deploy /var/www/*
# /var/www/../../etc/shadow  matches.
```

**Files in `/etc/sudoers.d/` with a dot in the name are ignored**
```bash
/etc/sudoers.d/deploy.conf     # SILENTLY IGNORED
/etc/sudoers.d/deploy          # loaded
```

**`usermod -L` doesn't disable SSH key auth** — it only prefixes the hash with `!`. Remove `authorized_keys`.

**Editing `/etc/sudoers` without `visudo`** — one syntax error and sudo refuses to run at all. With root's password disabled, you're locked out.

**`su` without `-`** inherits your `PATH` — a trojan in your `~/bin` runs as root.

**Deleting a user leaves their files** — `userdel` without `-r` orphans the home dir, and a later user with the same UID inherits it.

**`cat /etc/passwd` on an LDAP box** shows a fraction of the users. Use `getent passwd`.

**Assuming `root` means UID 0 by name** — `awk -F: '$3==0' /etc/passwd` is the real check.

**PAM `sufficient` ordering** — a `sufficient` that succeeds skips everything below it, including your 2FA.

## 9. Where AI helps (and where it hurts)

- **Helps:** explaining a PAM stack line by line (genuinely good — the syntax is opaque and it's read a lot of it); generating `useradd` invocations with the right flags for a service account; explaining RUID/EUID/SUID; drafting an offboarding checklist.
- **Hurts:** AI writes **dangerously permissive `sudoers`** by default — ask for "let deploy restart the service" and you'll get `deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp`, with no mention of the pager escape or the wildcard risk, because that's what every blog post says. It suggests `usermod -G` without `-a` often enough to matter. It will tell you `usermod -L` disables an account. And it does **not** reliably flag that a rule is GTFOBins-escapable — you have to ask. **Never apply AI-generated `sudoers` or PAM config without checking it against GTFOBins and `visudo -c`.** Ask explicitly: *"can this binary spawn a shell?"* and *"what's the full trust chain for this rule?"*

## 10. Learn independently

**Books (specific chapters):**
- *The Linux Programming Interface* (Kerrisk) — **Ch. 8 "Users and Groups"** (the passwd/shadow/group files, `getpwnam`, crypt), **Ch. 9 "Process Credentials"** (real/effective/saved IDs, setuid semantics, `seteuid` vs `setuid` — the definitive treatment; read it twice), and **Ch. 39 "Capabilities"** (the setuid decomposition). Kerrisk's Ch. 9 is the reference for everything in §2.2 and §6.
- *UNIX and Linux System Administration Handbook*, 5th ed. (Nemeth et al.) — **Ch. 8 "User Management"** and **Ch. 27 "Security"** — the practical operator's view: account lifecycle, `sudo` policy design, and PAM in production. Nemeth is the book to read for *how to run this at an organisation*, as opposed to how it works.
- *How Linux Works*, 3rd ed. (Ward) — **Ch. 7 "System Configuration"** (§7.3 users and passwords, §7.4 PAM) — the clearest short explanation of the PAM stack model anywhere.
- *The Linux Command Line* (Shotts) — **Ch. 9 "Permissions"** (covers `su`/`sudo` and the identity basics from the shell side).
- *Site Reliability Engineering* (Google, free at [sre.google/books](https://sre.google/books/)) — the chapters on **access control and change management** for the "why zero-standing-privilege?" argument. Its companion, *Building Secure and Reliable Systems*, has an excellent chapter on **least privilege** that is essentially this lesson's philosophy at Google scale — also free online.

**Primary docs (man pages):**
- [`man 7 credentials`](https://man7.org/linux/man-pages/man7/credentials.7.html) — **the** normative source for real/effective/saved IDs. Short; read it.
- [`man 5 passwd`](https://man7.org/linux/man-pages/man5/passwd.5.html), [`man 5 shadow`](https://man7.org/linux/man-pages/man5/shadow.5.html), [`man 5 group`](https://man7.org/linux/man-pages/man5/group.5.html)
- [`man 5 sudoers`](https://www.sudo.ws/docs/man/sudoers.man/) — long, but the **SUDOERS OPTIONS** and **Command Aliases** sections are essential. The security notes at the end are excellent.
- [`man 8 sudo`](https://www.sudo.ws/docs/man/sudo.man/) and [`man 8 visudo`](https://www.sudo.ws/docs/man/visudo.man/)
- [`man 5 pam.conf`](https://man7.org/linux/man-pages/man5/pam.conf.5.html) and [`man 8 pam`](https://man7.org/linux/man-pages/man8/pam.8.html) — plus `man 8 pam_unix`, `man 8 pam_limits`, `man 8 pam_faillock` for the individual modules.
- [`man 7 capabilities`](https://man7.org/linux/man-pages/man7/capabilities.7.html) — the full list; read what `CAP_SYS_ADMIN` actually covers and be alarmed.
- [`man 5 nsswitch.conf`](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html) — the NSS half of the NSS/PAM distinction.

**High-quality links:**
- [**GTFOBins**](https://gtfobins.github.io/) — **bookmark this permanently.** Before you write any `sudoers` rule, look up the binary. It is the single most useful link in this lesson.
- [The Linux-PAM System Administrators' Guide](http://www.linux-pam.org/Linux-PAM-html/Linux-PAM_SAG.html) — the official, complete PAM documentation.
- [sudo.ws — Security Policy and Sudoers](https://www.sudo.ws/docs/) — Todd Miller's own docs; the security considerations section is written by the person who knows.
- [Google, *Building Secure and Reliable Systems*, Ch. 5 "Design for Least Privilege"](https://google.github.io/building-secure-and-reliable-systems/raw/ch05.html) — free, and the best available writing on *why* the practices in §7 exist.
- [CVE-2021-4034 (PwnKit) analysis](https://blog.qualys.com/vulnerabilities-threat-research/2022/01/25/pwnkit-local-privilege-escalation-vulnerability-discovered-in-polkits-pkexec-cve-2021-4034) — read one setuid CVE writeup end to end; it makes §2.2 visceral.

## 11. Interview preparation

**Q1. What does the setuid bit do?**
> When you exec a binary with the setuid bit set, the kernel sets the process's **effective UID** to the file's **owner**, while the **real UID** stays the invoking user. So `/usr/bin/passwd` — owned by root, mode `4755` — runs with EUID 0 (needed to write `/etc/shadow`, which is `640 root:shadow`) while its RUID is still yours, which is how it knows *whose* password to change. That's the elegance of the design: **RUID says who you are, EUID says what you can do.** The third ID, the saved UID, lets a privileged process drop to the real user for risky work and regain privilege afterwards. The security consequence is that every setuid-root binary is attacker-controlled input running as root — historically the richest local-privilege-escalation surface on Linux (PwnKit, Baron Samedit) — which is why modern practice replaces them with fine-grained **capabilities** (`cap_net_raw` on `ping` instead of setuid-root) and audits `find / -perm -4000` against a baseline.

**Q2. Why are there two files, `/etc/passwd` and `/etc/shadow`?**
> `/etc/passwd` must be **world-readable** so any process can map UIDs to names — that's what makes `ls -l` show "ann" instead of 1001. Originally the password hashes lived there, which meant every user on the system could read every hash and crack them offline at leisure. Splitting the hashes into `/etc/shadow`, mode `640 root:shadow`, removed that. The `x` in the password field of `passwd` is the marker meaning "the real hash is in shadow." It's a good example of a security boundary retrofitted onto a design that predated the threat model.

**Q3. `sudo` vs `su`?**
> `su` requires **root's password** and gives all-or-nothing access; `sudo` requires **your own password** and grants specific commands. Every axis that matters favours `sudo`: **no shared secret** (with `su`, five admins share one password, and when one leaves you rotate it for everyone — and they still know the old one worked); **audit trail** (`sudo` logs who ran what, when, from where — `su` logs only "someone became root"); **granularity** (per-command, per-host, per-user); and **revocation** (delete a line vs. rotate a secret for everyone). Modern practice disables root's password entirely so `su` to root is impossible. One extra detail worth mentioning: `su` without `-` inherits **your** `PATH`, so a trojan in a writable `~/bin` runs as root — `sudo` defends against exactly this by default with `env_reset` and `secure_path`.

**Q4. Why is `sudo vim` dangerous?**
> Because `vim` can spawn a shell — `:!sh` — and that shell inherits vim's root privileges. So `deploy ALL=(ALL) NOPASSWD: /usr/bin/vim` is exactly equivalent to `(ALL) ALL`. The general rule is that **any binary that can spawn a subprocess, write an arbitrary file, or invoke a pager is root**: `less`, `find -exec`, `awk 'BEGIN{system()}'`, `tar --to-command`, `git -c core.pager`, `cp` (copy over `/etc/sudoers`), `systemctl` (it pages through `less`). [GTFOBins](https://gtfobins.github.io/) catalogues hundreds. So reviewing a `sudoers` rule means asking "can this program be made to execute something else or write anywhere?", not "does this sound dangerous?" The correct pattern is a purpose-built wrapper script taking no arguments, root-owned and **not writable by the caller** — because if the caller can edit the script, the rule is full root anyway.

**Q5. What's the difference between real, effective, and saved UID?**
> **Real** = who started the process (used for signal permission checks and accounting). **Effective** = what the kernel checks for access decisions — the one that actually matters. **Saved** = a stash of the effective UID from exec time, so a privileged process can drop privileges temporarily and regain them. Normally all three are identical; setuid execution makes EUID = the file owner while RUID stays the invoker. The saved UID is what makes `seteuid(getuid()); do_risky_thing(); seteuid(saved);` possible — without it, a process that dropped privilege could never get it back, so it would have to do *everything* as root, which defeats the purpose. You can see all three with `ps -o pid,ruid,euid,suid`.

**Q6. Explain PAM's control flags.**
> `required` — on failure, **remember it but keep going**, and fail at the end of the stack. `requisite` — on failure, **return immediately**. `sufficient` — on **success**, the stack succeeds immediately (unless a prior `required` already failed); on failure, ignore and continue. `optional` — ignored, unless it's the only module. The subtle one is why `required` fails *late*: it prevents an attacker from learning **which** check failed from the timing or the error message, which would leak whether a username exists. `requisite` fails fast where there's no information to protect. And the dangerous one is `sufficient`: a `sufficient` module that succeeds **skips everything below it** — so a single misordered line can bypass your 2FA module entirely, and logins will still work, so it looks fine. That's why PAM edits go through review and why you test with `pamtester` before trusting your SSH session to them.

**Q7. What's the difference between PAM and NSS?**
> Different questions. **NSS** (`/etc/nsswitch.conf`) is identity **lookup**: "who is UID 1001?", "does the user `ann` exist?", "what groups is she in?" — used by `getent`, `id`, `ls -l`. **PAM** is **authentication**: "can this person prove they're ann?" A user can exist via NSS (from LDAP) but fail PAM (wrong password); or authenticate via PAM but have no NSS entry, in which case they have no UID, no home, and no shell — a genuinely bewildering failure. The practical value is in triage: "user exists but can't log in" → PAM; "login works but `id` says unknown user" → NSS. Also why you should use `getent passwd` rather than `cat /etc/passwd` — `getent` goes through NSS and sees the directory users the file doesn't have.

**Q8. (System design) Design SSH access for a 50-person engineering org across 200 servers.**
> The goal is that access is **derived from identity, expires automatically, and offboarding is one action**. I'd avoid managing `authorized_keys` at all — it's the thing that structurally can't be offboarded reliably, because keys are copied and you never really know where they are. So: **short-lived SSH certificates** signed by a CA that authenticates against the company IdP (Netflix's BLESS, Teleport, or Vault's SSH CA). Servers trust the CA, not individual keys; certificates live 8 hours; revocation is "stop issuing," which is automatic when someone's IdP account is disabled. **A bastion** as the single audited entry point, with session recording. **Individual accounts** (or better, no local accounts — SSSD-backed or provisioned just-in-time by the cert's principals), never a shared `ubuntu` user, because you cannot answer "who deployed at 3 a.m." otherwise. **No standing sudo** — elevation is requested, approved, time-boxed, and logged to the SIEM, with alerting on break-glass use. **Break-glass**: a sealed local root credential in a vault, because if the IdP is down nobody can log in and you need a way back — that's a real failure mode people forget. `PasswordAuthentication no` and `PermitRootLogin no` in `sshd_config` throughout. The honest caveat: this is a lot of machinery for 50 people, so I'd stage it — individual accounts + centralised `authorized_keys` via config management + `sudo` with per-command rules gets you 80% of the value in a week, and the CA is the next step when the key-sprawl pain justifies it.

**Q9. (Trade-off) Why not just run the service as root? It's in a container anyway.**
> Because the container isn't the boundary people think it is. Without **user namespaces**, container UID 0 **is** host UID 0 — so any container escape (a kernel bug, a bad bind mount, a `--privileged` flag someone added to debug something) gives root on the host. With `USER app`, an escape gives you an unprivileged user instead, which is a completely different incident. The same argument applies inside the container: a compromised root process can install packages, modify binaries, and write anywhere, whereas an unprivileged one can't tamper with its own code or config — which is why config should be root-owned and app-readable, not app-owned. It's defence in depth: none of these individually stops a determined attacker, but each one converts a total compromise into a contained one. The counter-argument is real — non-root containers are more annoying (port <1024 needs `cap_net_bind_service`, volume permissions get fiddly, some images assume root) — but the fix for each is well-trodden, and "it's isolated anyway" is the reasoning that shows up in every container-escape post-mortem.

**Q10. How do you properly offboard a user?**
> Four steps, and people reliably do three. **(1)** Lock the password: `usermod -L` (prefixes the hash with `!`). **(2)** Set a nologin shell: `usermod -s /usr/sbin/nologin`. **(3)** Expire the account: `usermod -e 1`. **(4)** **Remove `~/.ssh/authorized_keys`** — and this is the one that matters, because `usermod -L` only locks *password* authentication; SSH key login still works perfectly. Then: kill their running processes (`pkill -u`), check for their cron jobs and systemd user units, remove them from `sudoers` and any groups, revoke their certificates/API tokens, and archive rather than delete their home directory — because `userdel -r` throws away data you may need, and a later user with the same UID would inherit anything left behind. The meta-answer is that this checklist is exactly why you want centralised identity and short-lived credentials: with a directory or a CA, offboarding is *one* action and it can't be half-done.

**Q11. Someone is in the `docker` group. What have you granted them?**
> Root. `docker run -v /:/host -u 0 -it alpine chroot /host sh` gives them a root shell on the host filesystem — the Docker daemon runs as root and will happily mount anything anywhere for anyone who can talk to its socket. Group membership *is* socket access, and socket access *is* root. So `docker` group is not a convenience permission, it's a root grant with a friendly name, and it should be modelled that way in your access control and flagged in any audit. The mitigations: **rootless Docker** (the daemon runs as the user), **Podman** (daemonless, rootless by default), or gating access behind `sudo` on specific wrapper scripts. The same reasoning applies to anything that talks to a root daemon's socket — the socket is the privilege boundary, not the group name.

**Q12. (Senior) You're reviewing a `sudoers` file. What's your checklist?**
> I'd work outward from each rule's trust closure rather than reading it line by line. **Per rule:** is the target binary GTFOBins-escapable (can it spawn a shell, write arbitrary files, or invoke a pager)? Are there wildcards — and do I realise `*` doesn't stop at `/`, so `restart myapp*` matches `myapp../../../bin/sh`? Is `NOPASSWD` justified (automation, yes; humans, rarely)? Does the rule specify a `runas` user, or is it `(ALL)` when it could be `(root)` or narrower? **Per referenced file:** is the wrapper script root-owned and **not writable** by the caller or any group they're in — because if they can edit it, the rule is `(ALL) ALL`. Is the same true of every file *that* script sources or executes? **Per file in the closure:** is anything in `/etc/sudoers.d/` group- or world-writable? Do any filenames contain a `.` (silently ignored, so the rule you think exists doesn't)? **Globally:** is `env_reset` still on and `secure_path` set? Are there `env_keep` entries that shouldn't be there (`LD_PRELOAD` and `PYTHONPATH` being the classics)? Does `awk -F: '$3==0' /etc/passwd` return exactly one line? And I'd verify the effective policy with `sudo -l -U <user>` per user rather than trusting my reading of the aliases, because alias expansion is where intent and reality drift apart. The framing I'd lead with: **auditing sudoers means auditing the transitive closure of everything a rule can reach**, not reading one file.

## 12. Homework

> Work in `linux/solutions/lesson-05/`. **Use a throwaway VM.** Several tasks create real privilege-escalation paths, and one asks you to lock yourself out. Snapshot first.

**Easy**

1. **Identity inventory.** Write `whoami-really.sh` printing: your UID/GID, real vs effective IDs, primary group, all supplementary groups, whether you're in `sudo`/`wheel`/`docker`, your shell, and your `sudo -l` output. Then run it under `sudo` and diff.
   *Acceptance:* the script plus a `NOTES.md` explaining every field that changed under `sudo` and why.

2. **Read the account files.** Without `cat`ting them raw, produce: every user with a login shell; every system user (UID < 1000); every account with a locked password; every account with **no** password; the hash algorithm in use per user; and every account with UID 0.
   *Acceptance:* one script using `getent` + `awk`; a written note on why `getent` beats `cat` and what the UID-0 check would mean if it returned two rows.

3. **Service account.** Create a `myapp` system account per §4.4 with proper directory ownership. Prove: it can't log in; it can read its config but not write it; it can write its data dir; it can't read `/etc/shadow`. Then explain in one paragraph why config is root-owned.
   *Acceptance:* the setup script plus four demonstrated proofs with output.

**Medium**

4. **Setuid, hands on.** Reproduce §4.1: build the RUID/EUID demo, make it setuid, observe. Then make a setuid-root copy of `bash` and get a root shell with it — and explain why `-p` is needed. Clean up. Finally, baseline all setuid binaries on the box, add one, and detect it with a diff.
   *Acceptance:* the full transcript, an explanation of bash's `-p` behaviour and why bash drops privileges by default, and a working `setuid-audit.sh` that diffs against a baseline file.

5. **Break `sudo` safely.** On a snapshot: introduce a syntax error in `/etc/sudoers` **without** `visudo` (e.g. `echo 'garbage' | sudo tee -a /etc/sudoers`). Observe that sudo now refuses everything. Recover three ways: an already-open root session, `pkexec`, and single-user mode / a rescue boot. Then show `visudo -c` catching the same error.
   *Acceptance:* evidence of the breakage and **all three** recoveries documented as runbooks. This teaches the lesson permanently.

6. **GTFOBins in practice.** Grant yourself `NOPASSWD` sudo on **five** binaries that sound harmless — pick from `find`, `awk`, `less`, `tar`, `git`, `cp`, `systemctl`, `env`, `nice`, `zip`. Escalate to root using each. Then write the safe equivalent for the *actual* task each was meant to do.
   *Acceptance:* five working escalations with the exact commands, and five safe wrapper-script alternatives with `sudoers` rules. Then rewrite §4.3's three attempts yourself and explain each flaw.

7. **Wildcards are worse than they look.** Create the rule `deploy ALL=(ALL) NOPASSWD: /bin/chown deploy /var/www/*` and escalate via path traversal. Then do the same with `/usr/bin/systemctl restart myapp*`. Explain precisely why sudo's `*` doesn't stop at `/`.
   *Acceptance:* two working escalations, an explanation citing `man 5 sudoers`' wildcard section, and a fixed rule for each.

**Hard**

8. **PAM: add 2FA without locking yourself out.** Install `libpam-google-authenticator` and add TOTP to SSH. Requirements: **keep a second session open**; test with `pamtester` *before* trusting it; make key-only logins still work (`AuthenticationMethods publickey,keyboard-interactive:pam`); exempt one break-glass account. Then deliberately misorder a `sufficient` line so 2FA is bypassed, and show that logins still succeed — proving the bypass is invisible.
   *Acceptance:* working 2FA, a documented rollback, the `pamtester` transcript, and the bypass demonstration with an explanation of the control-flag mechanics that caused it. **The bypass demo is the point of this task.**

9. **`pam_faillock` + audit.** Configure account lockout after 5 failed SSH attempts with a 10-minute unlock. Verify with a scripted brute-force from another host. Show `faillock --user X` reporting, and unlocking. Then configure `auditd` to watch `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, and `/etc/sudoers.d/`, and demonstrate the audit records for a modification.
   *Acceptance:* lockout working with evidence, the `faillock` output, the `auditd` rules, and `ausearch` output showing who changed what. Plus a note on the DoS risk of account lockout (an attacker can lock out your admins on purpose) and how you'd mitigate it.

10. **Capabilities instead of setuid.** Write a small server binding port 80. Run it four ways: (a) as root, (b) setuid-root, (c) unprivileged with `setcap cap_net_bind_service=+ep`, (d) unprivileged behind a systemd socket-activated unit. For each, inspect `getpcaps` and the process credentials, and assess: what does an RCE in this binary get an attacker?
    *Acceptance:* four working setups, a `getpcaps`/`ps -o ruid,euid` comparison table, and a written threat assessment per option with a recommendation. Then look up what `CAP_SYS_ADMIN` covers and write a paragraph on why "just add CAP_SYS_ADMIN" is not a small ask.

**Stretch (senior)**

11. **Audit a deliberately broken box.** Have a partner (or a script you write and then forget the details of — commit it and check out a branch without it) plant **eight** privilege-escalation paths from this list: a writable `sudoers.d` file, a `sudoers` rule on an escapable binary, a world-writable wrapper script referenced by a rule, an extra UID 0 account, a setuid-root shell, a user in `docker`, a stale `authorized_keys` for a deleted employee, a `sudoers.d` file with a `.` in the name masking a rule someone thinks exists, `env_keep` including `LD_PRELOAD`, a cron job running a user-writable script as root. Then find them all with a script.
    *Acceptance:* an `audit.sh` finding all eight (verify against the plant list), with output formatted as findings + severity + remediation. Compare your results against `LinPEAS` or `linux-smart-enumeration` and note what it caught that you didn't — and vice versa.

12. **Zero-standing-privilege prototype.** Build a working just-in-time sudo system: a user has **no** sudo by default; they run `request-sudo "reason"`, which writes a request to a log and (for the exercise, auto-)approves it by writing a time-limited `/etc/sudoers.d/` drop-in; a systemd timer revokes it after 15 minutes; every grant, use, and revocation is logged. Handle the edge cases: what if the timer doesn't fire? What if they're mid-command at expiry? What if the grant script itself is compromised?
    *Acceptance:* the working system; a demonstration of grant → use → auto-revoke; and a `THREATS.md` analysing the trust chain — specifically, who can write the grant script, who can write the timer unit, and what happens if the audit log is on the same box being compromised. The threat analysis is worth more than the implementation.

## 13. Key takeaways

- **The kernel knows numbers.** UIDs, not names. A deleted user's UID reassigned to someone else transfers file ownership silently — and `root` means **UID 0**, not the name, so `awk -F: '$3==0' /etc/passwd` is the real check.
- Every process has **real** (who started it), **effective** (what the kernel checks), and **saved** (the stash for temporary drops) UIDs. **Setuid sets EUID to the file's owner while RUID stays yours** — that's how `passwd` writes `/etc/shadow` yet knows whose password to change.
- **Every setuid-root binary is attack surface.** Audit `find / -perm -4000`; prefer **capabilities** (`setcap`) where you control the binary.
- `/etc/passwd` is world-readable (UID→name mapping), so hashes live in `/etc/shadow` (`640 root:shadow`). Hash format is `$id$salt$hash`; **yescrypt (`$y$`) is the modern default** and it's memory-hard on purpose.
- **`usermod -aG`. Never bare `-G`.** And group changes need a re-login (or a **service restart** for daemons).
- **`sudo` over `su`** — your password, per-command, audited, revocable, no shared secret. Disable root's password.
- **`sudoers` rules on escapable binaries are full root** — `vim`, `less`, `find`, `awk`, `tar`, `git`, `systemctl`, `cp`. Check **GTFOBins** before writing any rule. Wildcards don't stop at `/`. **Wrapper scripts: no arguments, root-owned, not writable by the caller.**
- **Auditing `sudoers` means auditing the transitive closure** of every file a rule can reach — including who can modify the wrapper.
- **PAM = authentication** (can you prove it?), **NSS = lookup** (who is this?). PAM's four groups × four control flags; a misordered **`sufficient` silently bypasses everything below it**. Always keep a second root session open, and test with `pamtester`.
- **`usermod -L` locks the password, not the account** — SSH keys still work. Offboarding is lock + nologin + expire + **remove `authorized_keys`**.
- **`docker` group = root.** Container UID 0 = host UID 0 without user namespaces. `USER app`, always.
