# Linux Lesson 02 — Files, Permissions & Ownership

| | |
|---|---|
| **Track** | Linux |
| **Lesson** | 02 of 15 |
| **Topic** | Managing files and directories, and the Unix permission/ownership model |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lesson 01 (shell navigation) |

### Learning objectives
1. Create, copy, move, remove, and link files and directories confidently (and safely).
2. Read `ls -l` output completely: type, permissions, owner, group, size, time.
3. Explain and set permissions with both **symbolic** and **octal** notation.
4. Manage ownership (`chown`/`chgrp`) and understand user/group/other.
5. Understand special bits (setuid, setgid, sticky) and `umask` at a working level.

---

## 1. Why this matters
Permissions are where security meets daily work. "Permission denied," a world-writable secret, an SSH key that's "too open," a deploy script that can't execute — all are permission issues. As a backend/DevOps engineer you'll set permissions correctly dozens of times a week, and getting them wrong is a real security incident. This is foundational and heavily tested in interviews.

---

## 2. Theory

### 2.1 Files, directories, and links
- A **file** holds data; a **directory** is a special file listing other files (name → inode).
- An **inode** stores a file's metadata (permissions, owner, size, timestamps, data block pointers) — *not* its name.
- **Hard link**: another name pointing to the same inode (same data; equal status). `ln target name`.
- **Symbolic (soft) link**: a small file pointing to a *path*; can cross filesystems and link directories; breaks if the target moves. `ln -s target name`.

### 2.2 File management commands
```bash
mkdir -p a/b/c          # create nested dirs (-p = make parents, no error if exists)
touch file.txt          # create empty file / update timestamp
cp src dst              # copy; cp -r for directories; cp -a to preserve attrs
mv src dst              # move OR rename
rm file                 # remove (no recycle bin!); rm -r dir; rm -rf force-recursive
rmdir dir               # remove empty dir
ln -s /path/target lnk  # symlink
```
**`rm` is permanent.** There's no trash. `rm -rf /` (or a bad variable expansion like `rm -rf "$DIR"/` with empty `$DIR`) is catastrophic.

### 2.3 Reading `ls -l`
```
-rw-r--r--  1 ann  devs  4096 Jun 28 10:00 report.txt
│└┬┘└┬┘└┬┘  │  │    │      │      │           └ name
│ │  │  │   │  │    │      │      └ modification time
│ │  │  │   │  │    │      └ size (bytes)
│ │  │  │   │  │    └ group
│ │  │  │   │  └ owner (user)
│ │  │  │   └ hard-link count
│ │  │  └ permissions for OTHER (everyone else)
│ │  └ permissions for GROUP
│ └ permissions for OWNER (user)
└ file type: - file, d dir, l symlink, c/b device, s socket, p pipe
```

### 2.4 The permission model: u/g/o × r/w/x
Three classes — **user (owner)**, **group**, **other** — each with three bits:
- **r (read)** = 4, **w (write)** = 2, **x (execute)** = 1.
- On a **file**: r=read contents, w=modify, x=run as program.
- On a **directory**: r=list names, w=create/delete entries, **x=enter/traverse** (you need `x` to `cd` into it or access files within, even if you have `r`).

### 2.5 Octal and symbolic notation
Sum the bits per class:
- `rwx` = 7, `rw-` = 6, `r-x` = 5, `r--` = 4, `--x`=1, `---`=0.
- `chmod 644 file` → `rw-r--r--` (owner rw, group r, other r) — typical for files.
- `chmod 755 dir`  → `rwxr-xr-x` — typical for directories & executables.
- `chmod 600 secret` → owner-only read/write — typical for keys/secrets.

Symbolic:
```bash
chmod u+x script.sh        # add execute for owner
chmod g-w file             # remove write for group
chmod o= file              # remove all perms for other
chmod a+r file             # add read for all (a = u+g+o)
chmod -R u+rwX dir         # recursive; capital X = x only on dirs/already-exec files
```

### 2.6 Ownership
```bash
chown ann file             # change owner
chown ann:devs file        # change owner and group
chgrp devs file            # change group only
chown -R app:app /srv/app  # recursive (deploy dirs)
```
Only **root** (or the owner, in limited ways) can change ownership. Changing owner usually needs `sudo`.

### 2.7 umask and default permissions
New files don't start at 777. The **umask** subtracts permission bits from the base (666 for files, 777 for dirs). A typical `umask 022` → new files `644`, new dirs `755`.

### 2.8 Special bits (know they exist)
- **setuid (4xxx)** on an executable: it runs as the *file owner* (e.g., `passwd` runs as root). Security-sensitive.
- **setgid (2xxx)** on a directory: new files inherit the directory's group (handy for shared project dirs).
- **sticky bit (1xxx)** on a directory (e.g., `/tmp` → `rwxrwxrwt`): only a file's owner can delete it, even in a world-writable dir.

---

## 3. Official documentation quotes

> "Each file has a set of permissions that control the kinds of access that users have to that file. … there are three kinds of permissions: read, write, and execute. The permissions for a file are also called its access modes."
> — *GNU Coreutils manual*, [File permissions](https://www.gnu.org/software/coreutils/manual/html_node/File-permissions.html)

> "chmod changes the file mode bits of each given file according to mode, which can be either a symbolic representation of changes to make, or an octal number representing the bit pattern for the new mode bits."
> — *chmod(1) man page* / GNU Coreutils

> "On directories, the execute permission (also called the 'search' bit) allows the affected user to enter the directory, and access files and directories inside."
> — paraphrasing *coreutils* docs; see also the `chmod(1)` and `path_resolution(7)` man pages.

---

## 4. Real-world examples

### 4.1 Set up a deploy directory correctly
```bash
sudo mkdir -p /srv/app
sudo chown -R deploy:deploy /srv/app   # app user owns it
sudo chmod 755 /srv/app                # owner full, others read+traverse
chmod 600 /srv/app/.env                # secrets: owner-only
chmod +x /srv/app/run.sh               # make the launcher executable
```

### 4.2 Diagnose "Permission denied"
```bash
ls -l run.sh
# -rw-r--r-- 1 ann devs 120 ... run.sh   → no x bit!
./run.sh                  # Permission denied
chmod u+x run.sh          # fix
./run.sh                  # works
```

### 4.3 The SSH key gotcha
```bash
chmod 600 ~/.ssh/id_ed25519     # private key MUST be owner-only
chmod 644 ~/.ssh/id_ed25519.pub
chmod 700 ~/.ssh                # directory owner-only
# If too open, ssh refuses: "UNPROTECTED PRIVATE KEY FILE!"
```

### 4.4 Symlink vs hard link
```bash
echo "data" > original.txt
ln original.txt hard.txt        # same inode (ls -i shows equal inode numbers)
ln -s original.txt soft.txt     # points to the path
rm original.txt
cat hard.txt                    # still works (data persists via the other name)
cat soft.txt                    # broken: No such file or directory
```

---

## 6. How This Is Used In Production
- **Startups:** correct ownership/permissions on deploy dirs, config, and secrets are part of every deployment. The classic incident: a `.env` or credentials file left world-readable (`644` instead of `600`).
- **Enterprises:** least-privilege enforced via dedicated **service accounts** (`app:app`, never root), setgid shared directories for teams, and config management (Ansible/Chef) that asserts exact modes/owners on every run. Audits flag world-writable files and stray setuid binaries.
- **Common architectures:** apps run as unprivileged users; data dirs owned by that user; secrets mounted read-only with tight modes (or injected via a secrets manager); containers run as non-root (`USER app` in the Dockerfile) so a compromise doesn't get root.
- **Scaling:** permissions are declared in config management / Dockerfiles / Kubernetes securityContext so thousands of hosts/containers are identical and verifiable, not hand-set.
- **Monitoring/Logging:** log dirs are writable by the app user, readable by the log shipper; wrong perms = "logs stopped." Security scanners (Lynis, CIS benchmarks) check for over-permissive files and unexpected setuid bits.
- **Security:** this *is* a security topic — over-permissive files, world-writable directories, and unnecessary setuid binaries are common attack vectors. Principle: grant the *minimum* needed.
- **Real examples:** SSH refusing keys with loose permissions is a daily reality at every company using **GitHub**/cloud VMs. Container hardening guidance from **Google** (distroless), **Docker**, and the **CIS Benchmarks** centers on non-root users and least-privilege file modes — exactly this lesson, at scale.

---

## 7. Best practices
- **Least privilege:** give the narrowest permissions that work. Files `644`, dirs `755`, secrets `600`, private keys `600` in a `700` dir.
- Run apps as a **dedicated non-root user**; own their files with that user.
- Never `chmod 777` to "make it work" — it's almost always wrong and a security hole.
- Be terrified of `rm -rf` with variables; quote and validate (`[ -n "$DIR" ]`) first.
- Prefer setting perms declaratively (Dockerfile, Ansible) over manual `chmod` in prod.
- Use `cp -a`/`rsync -a` when you need to preserve permissions/ownership/timestamps.

## 8. Common mistakes & gotchas
- Forgetting directory **`x`** — you have `r` but still can't `cd`/access files inside.
- `chmod 777` as a lazy fix — opens write to the world.
- `rm -rf "$VAR/"` when `$VAR` is empty/unset → targets `/`. Disaster.
- Editing a file you don't own and getting "Permission denied," then `sudo`-editing and accidentally making root the owner of your file.
- Assuming a symlink behaves like a copy — deleting the target breaks it.
- Recursive `chmod -R 644` on a tree removes `x` from directories, making them inaccessible — use capital `X` (`chmod -R u+rwX`).

## 9. Where AI helps (and where it hurts)
- **Helps:** converting between octal and symbolic, explaining an `ls -l` line, diagnosing "Permission denied," recommending correct modes for secrets/keys/deploy dirs, explaining setuid/sticky.
- **Hurts:** AI sometimes suggests `chmod 777` or `sudo` as quick fixes, or recursive `chmod` that breaks directory traversal. It also can't see your *actual* owner/mode — run `ls -l`/`id` and give it the output. Always sanity-check any recursive or `sudo` command it proposes.

## 10. Learn independently
- [GNU Coreutils — File permissions](https://www.gnu.org/software/coreutils/manual/html_node/File-permissions.html).
- `man chmod`, `man chown`, `man umask`, `man 7 path_resolution`.
- *The Linux Command Line* (Shotts), permissions chapters.
- CIS Benchmarks / Lynis to see what "secure permissions" looks like in practice.

## 11. Interview preparation

**Q1. What does `chmod 644` mean?**
> Owner read+write (6), group read (4), other read (4): `rw-r--r--`. The digits are octal sums of read(4)+write(2)+execute(1) for user/group/other.

**Q2. What does the execute bit do on a directory?**
> It's the "search/traverse" permission — you need it to enter the directory (`cd`) and access files inside, even to read a file you otherwise have permission to. Read without execute lets you list names but not use them.

**Q3. Difference between a hard link and a symbolic link?**
> A hard link is another directory entry pointing to the same inode/data — equal status, same filesystem, data persists until all links are gone. A symlink is a small file containing a path to the target; it can cross filesystems and link directories but breaks if the target is removed/moved.

**Q4. What's the difference between `chmod` and `chown`?**
> `chmod` changes permission bits (who can read/write/execute). `chown` changes ownership (which user/group owns the file). Changing ownership generally requires root.

**Q5. What is `umask`?**
> A mask of permission bits removed from the default when new files/dirs are created. With `umask 022`, new files become 644 and new directories 755 (base 666/777 minus 022).

**Q6. What is the setuid bit and why is it security-sensitive?**
> setuid makes an executable run with the privileges of its owner rather than the caller (e.g., `passwd` runs as root to edit `/etc/shadow`). It's risky because a vulnerable setuid-root binary can grant attackers root; unexpected setuid binaries are a red flag in audits.

**Q7. SSH says your private key is "too open." Why and how to fix?**
> SSH refuses private keys readable by group/other for security. Fix with `chmod 600 ~/.ssh/id_*` and `chmod 700 ~/.ssh`.

## 12. Homework
> Work in `linux/solutions/lesson-02/`.

**Easy**
1. Create `lab/` with a few files. Practice `cp`, `mv`, `mkdir -p`, and removing a file. Run `ls -l` and, for one file, explain every field of the output in a comment.

**Medium**
2. Create a script `hello.sh` that echoes something. Show it fails to run without the execute bit, then fix it with `chmod u+x`. Create a `secret.txt`, set it to `600`, and verify with `ls -l`. Create a directory, remove its `x` bit for "other," and demonstrate (as appropriate) the traverse restriction. Convert three different `ls -l` permission strings to octal by hand and verify with `stat -c '%a'`.

**Hard**
3. Simulate a deploy layout: `/tmp/srv/app` owned by a group, with `app/` at `755`, a `.env` at `600`, a `run.sh` executable, and a `logs/` directory writable by the app. Use `umask` to demonstrate default permissions on a newly created file, then change `umask` and show the difference. Document each command and the resulting `ls -l`. (Use `sudo`/`chown` only if you have rights; otherwise demonstrate `chmod`/`chgrp` within your own files and note where `chown` would be needed.)

**Stretch:** Demonstrate hard vs symbolic links: create both to a file, show equal inode numbers for the hard link (`ls -i`), delete the original, and show which link still works. Explain why.

## 13. Key takeaways
- `ls -l` encodes type + permissions (u/g/o × rwx) + owner/group + size + time.
- Permissions: r=4, w=2, x=1; common modes 644 (files), 755 (dirs/exec), 600 (secrets/keys).
- On directories, `x` means traverse/enter — required to access contents.
- `chmod` = permissions, `chown` = ownership; `umask` sets defaults.
- Least privilege always; never `chmod 777`; treat `rm -rf` with variables as dangerous.
- Hard links share an inode; symlinks point to a path and can break.
