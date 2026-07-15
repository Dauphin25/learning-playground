# Linux Lesson 06 — Package Management & Software Installation

| | |
|---|---|
| **Track** | Linux |
| **Lesson** | 06 of 15 |
| **Topic** | `dpkg`/`apt`, `rpm`/`dnf`, repositories, GPG trust, dependency resolution, and building from source |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–05 (filesystem, permissions, processes, users) |

### Learning objectives
1. Explain the **two-layer model** — low-level (`dpkg`/`rpm`) vs high-level (`apt`/`dnf`) — and why it exists.
2. Manage packages confidently on both families: query, install, remove, purge, hold, downgrade, and inspect.
3. Explain how **repositories and GPG signing** establish trust, and what `apt-key add` being deprecated was actually about.
4. Reason about **dependency resolution**, versioning, and why "dependency hell" is a genuinely hard problem.
5. Decide correctly between distro packages, language package managers, containers, Snap/Flatpak, and building from source.
6. Handle the real operational concerns: pinning, unattended upgrades, config file conflicts, and reproducibility.

---

## 1. Why this matters

Package management is how software gets onto a machine, and therefore it's where **supply chain security** lives. Every `apt install` is an act of trust: you're executing maintainer scripts as root, from a server, based on a signature chain you probably haven't examined. Understanding that chain is the difference between an engineer and someone who copies `curl | sudo bash` from a README.

For your target roles:
- **DevOps/platform:** you'll build Docker images (every `RUN apt-get install` is a reproducibility and image-size decision), manage patching across a fleet, pin versions to stop surprise upgrades, and answer "are we vulnerable to CVE-XXXX?" — which is a package-database query. You'll also debug the classic "works on my machine" that's actually a package-version difference.
- **Backend:** your `requirements.txt` and the OS package for `libpq-dev` are the same problem at different layers, and knowing why `pip install psycopg2` needs a compiler while `psycopg2-binary` doesn't is a real, daily distinction.
- **Interviews:** "`apt` vs `dpkg`?", "how does the system know a package is authentic?", "why not `curl | bash`?" and "what's dependency hell?" are standard. The `curl | bash` question is a values question as much as a technical one.

---

## 2. Theory

### 2.1 The two layers, and why

Every packaging system has exactly two layers, and confusing them causes most package-management pain:

| | **Low-level** | **High-level** |
|---|---|---|
| Debian family | `dpkg` | `apt` / `apt-get` |
| RHEL family | `rpm` | `dnf` (was `yum`) |
| Arch | — | `pacman` (both) |
| Alpine | — | `apk` (both) |
| SUSE | `rpm` | `zypper` |

**The low-level tool installs one file.** It knows how to unpack an archive, run maintainer scripts, and record what happened in a database. It **checks** dependencies but **cannot resolve** them — hand it a `.deb` needing `libfoo`, and it errors.

**The high-level tool resolves and fetches.** It knows about repositories, downloads, works out the transitive dependency graph, orders the installs, and then calls the low-level tool to do the actual work. `apt` is a dependency solver and downloader wrapped around `dpkg`.

```bash
$ sudo dpkg -i myapp_1.0_amd64.deb
dpkg: dependency problems prevent configuration of myapp:
 myapp depends on libpq5; however:
  Package libpq5 is not installed.
$ sudo apt install -f              # apt: "I'll fix that" — resolves and fetches libpq5
$ sudo apt install ./myapp_1.0_amd64.deb    # better: let apt handle the local file from the start
```
That interaction — `dpkg` complains, `apt install -f` fixes — is the two-layer model made visible, and it's the answer to "`apt` vs `dpkg`?"

### 2.2 What a package actually is

A `.deb` is an `ar` archive containing three members:
```bash
$ ar t nginx_1.24.0-1_amd64.deb
debian-binary          # format version, literally "2.0\n"
control.tar.xz         # METADATA + maintainer scripts
data.tar.xz            # THE FILES — a tarball rooted at /

$ dpkg-deb -c nginx_1.24.0-1_amd64.deb | head -3     # list contents
drwxr-xr-x root/root  0  ./
drwxr-xr-x root/root  0  ./etc/
-rw-r--r-- root/root  1077  ./etc/nginx/nginx.conf

$ dpkg-deb -e nginx_1.24.0-1_amd64.deb /tmp/ctl && ls /tmp/ctl
conffiles  control  md5sums  postinst  postrm  preinst  prerm  templates
```

`control` is the metadata:
```
Package: nginx
Version: 1.24.0-1
Architecture: amd64
Depends: libc6 (>= 2.34), libpcre2-8-0 (>= 10.22), libssl3 (>= 3.0.0)
Recommends: nginx-common
Suggests: nginx-doc
Conflicts: apache2
Replaces: nginx-light
Provides: httpd
```

The relationship fields, which are more expressive than people realise:
- **`Depends`** — hard requirement; won't install without it.
- **`Pre-Depends`** — must be *fully configured* before this even unpacks. Rare and serious.
- **`Recommends`** — installed **by default** (this surprises people — `--no-install-recommends` is why Docker images are bloated).
- **`Suggests`** — never auto-installed; just a hint.
- **`Conflicts`** / **`Breaks`** — can't coexist.
- **`Provides`** — a virtual package. Both `nginx` and `apache2` `Provide: httpd`, so anything depending on `httpd` accepts either. This is how alternatives work.
- **`Replaces`** — takes over files from another package.

**The maintainer scripts run as root**: `preinst` → unpack → `postinst` → (later) `prerm` → remove → `postrm`. **This is the crux of package trust**: installing a package means executing arbitrary shell as root. The signature chain (§2.4) is what makes that acceptable. It's also why "just add this third-party repo" is a bigger decision than it looks.

The RPM equivalent is a single binary file with a header + cpio payload, and scriptlets `%pre`/`%post`/`%preun`/`%postun`:
```bash
$ rpm -qip nginx-1.24.0-1.x86_64.rpm      # info
$ rpm -qlp nginx-1.24.0-1.x86_64.rpm      # file list
$ rpm -q --scripts nginx                  # SEE THE SCRIPTLETS — do this before trusting a package
$ rpm2cpio nginx.rpm | cpio -idmv         # extract without installing
```
`rpm -q --scripts` (and `dpkg-deb -e` for Debian) is how you actually inspect what a package will do to your system. Almost nobody does it; it's exactly what you'd do when evaluating a vendor package.

### 2.3 The package database

The low-level tool maintains a local database of what's installed and which file came from where.

```bash
# Debian
$ dpkg -l | head -5                        # everything installed
$ dpkg -l nginx                            # one package's status
ii  nginx  1.24.0-1  amd64  small, powerful, scalable web/proxy server
││
│└─ desired state: i=install, h=hold, r=remove, p=purge
└── actual state: i=installed, c=config-files only, u=unpacked, F=half-configured

$ dpkg -L nginx                            # what files does nginx own?
$ dpkg -S /usr/sbin/nginx                  # WHICH PACKAGE owns this file?
nginx-core: /usr/sbin/nginx
$ dpkg -V nginx                            # VERIFY — has anything been modified?
??5?????? c /etc/nginx/nginx.conf          # '5' = md5 mismatch, 'c' = it's a conffile

# RHEL
$ rpm -qa                                  # everything
$ rpm -ql nginx                            # file list
$ rpm -qf /usr/sbin/nginx                  # owner of a file
$ rpm -V nginx                             # verify
$ rpm -qi nginx                            # full info
$ rpm -q --changelog nginx | head          # CHANGELOG — this is how you check for a CVE fix
```

**`dpkg -S` / `rpm -qf` are the "where did this file come from?" tools**, and they're how you start any investigation into an unexpected binary. **`dpkg -V` / `rpm -V` are integrity checks** — they compare on-disk files against the package's recorded checksums, so they'll show you what's been modified since install. That's a poor man's file-integrity monitor and a real forensic tool: after a suspected compromise, `rpm -Va` tells you which system binaries no longer match their packages. (Caveat: an attacker with root can update the database too, so it's a first look, not proof.)

`rpm -q --changelog` deserves special mention: it's how you answer "is CVE-2024-XXXX fixed in the version we have?" without guessing from version numbers, because distros **backport** security fixes without bumping the upstream version (§6).

### 2.4 Repositories and trust — the part that matters

A repository is a directory tree of packages plus **signed** index files. The trust chain:

1. The repo publishes an index (`Release`/`Packages` for Debian, `repomd.xml` for RPM) listing every package with its **SHA256 hash**.
2. The index is **GPG-signed** by the repo's key (`Release.gpg`, or an inline-signed `InRelease`).
3. Your machine holds the **public key** and verifies that signature.
4. Package integrity then follows from the hash in the verified index.

**So the security rests entirely on the key. Everything else is a hash chain hanging off it.** That single sentence is the answer to "how does the system know a package is authentic?", and it explains everything below.

```bash
$ cat /etc/apt/sources.list.d/docker.list
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable
#    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ THIS is the important part
```

**Why `apt-key add` was deprecated** — this is the interview-worthy bit. Keys added with `apt-key` went into a **global trusted keyring**, meaning **any** key could sign for **any** repository. So adding a third-party repo's key (say, a random PPA) gave that key authority to sign a replacement `openssh-server` or `libc6` from the *official* Ubuntu repo. One sketchy PPA compromised your entire base system, silently. `signed-by=` scopes a key to exactly one repository, which is what everyone assumed `apt-key` did all along.

The modern pattern:
```bash
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
    | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
    | sudo tee /etc/apt/sources.list.d/docker.list
$ sudo apt update
```
Note that **HTTPS is not the security control here** — signing is. HTTPS hides *what* you're downloading from a network observer and stops trivial tampering, but the GPG signature is what proves authenticity, and it works over plain HTTP through a caching proxy. That's why Debian mirrors were historically HTTP and it wasn't (entirely) negligence. This is a great "explain the threat model" answer.

Deb822 is the newer, clearer format:
```bash
$ cat /etc/apt/sources.list.d/docker.sources
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: jammy
Components: stable
Signed-By: /etc/apt/keyrings/docker.gpg
```

RPM side:
```bash
$ cat /etc/yum.repos.d/docker-ce.repo
[docker-ce-stable]
name=Docker CE Stable
baseurl=https://download.docker.com/linux/centos/$releasever/$basearch/stable
gpgcheck=1                                    # NEVER set this to 0
gpgkey=https://download.docker.com/linux/centos/gpg
$ rpm -qa gpg-pubkey*                         # which keys are trusted?
$ rpm --import https://.../RPM-GPG-KEY-EPEL-9
```
`gpgcheck=0` in a repo file is an immediate finding. It means "install anything from anyone who can MITM this connection."

### 2.5 Daily operations

```bash
# --- Debian/Ubuntu ---
sudo apt update                       # refresh the INDEX. Does NOT upgrade anything.
sudo apt upgrade                      # upgrade installed packages; never removes anything
sudo apt full-upgrade                 # will REMOVE packages if needed to resolve — read it first
sudo apt install nginx=1.24.0-1       # a specific version
sudo apt install --no-install-recommends nginx     # skip Recommends — do this in Docker
sudo apt remove nginx                 # remove binaries; KEEP config
sudo apt purge nginx                  # remove binaries AND config
sudo apt autoremove                   # drop orphaned auto-installed deps
apt list --installed | grep nginx
apt-cache policy nginx                # WHICH versions are available, from WHERE, what's installed
apt-cache depends nginx ; apt-cache rdepends nginx    # forward and REVERSE dependencies
apt-mark hold nginx                   # PIN: never upgrade this
apt-mark showhold
sudo apt download nginx               # fetch the .deb without installing
apt changelog nginx                   # why did this version change?

# --- RHEL/Fedora ---
sudo dnf check-update
sudo dnf upgrade
sudo dnf install nginx-1.24.0
sudo dnf install --setopt=install_weak_deps=False nginx   # the Recommends equivalent
sudo dnf remove nginx
sudo dnf autoremove
dnf info nginx ; dnf repoquery --requires nginx ; dnf repoquery --whatrequires nginx
sudo dnf history                      # TRANSACTION LOG
sudo dnf history undo 42              # <-- ROLL BACK a transaction. apt cannot do this.
sudo dnf downgrade nginx
dnf provides */nginx.conf             # which package provides this file?
```

**`apt update` vs `apt upgrade` is the classic confusion**: `update` refreshes the *index* (it downloads the list of what's available and changes nothing on your system); `upgrade` installs newer versions. You always need `update` first, or `upgrade` acts on stale information. In `dnf`, `check-update`/`upgrade` are more honestly named.

**`apt-cache policy` is the most underrated command here.** It tells you, in one shot: what's installed, what's the candidate for install, and every available version with its source and pin priority. It's the first thing to run when "apt won't install the version I want."

**`dnf history undo` has no `apt` equivalent**, and it's a genuine RHEL advantage — a bad upgrade can be reversed as a transaction. On Debian you'd need to manually reinstall the old versions from `/var/cache/apt/archives`, if they're still there.

### 2.6 Config file handling — the operational trap

Packages mark certain files as **conffiles**, and the system tries very hard not to clobber your edits:

```bash
$ dpkg-query --showformat='${Conffiles}\n' --show nginx
 /etc/nginx/nginx.conf 8f4e2b... 
```

On upgrade, `dpkg` compares three things: the original shipped file, your current file, and the new shipped file.
- You didn't modify it → silently replaced.
- You modified it, package didn't change it → keep yours.
- **Both changed → prompt.** This is where automation hangs forever.

```
Configuration file '/etc/nginx/nginx.conf'
 ==> Modified (by you or by a script) since installation.
 ==> Package distributor has shipped an updated version.
   What would you like to do about it?
    Y or I  : install the package maintainer's version
    N or O  : keep your currently-installed version
      D     : show the differences
      Z     : start a shell
```

**In automation this prompt blocks forever** and your CI job or Ansible run hangs until it times out. The fix:
```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get -y \
  -o Dpkg::Options::="--force-confdef" \
  -o Dpkg::Options::="--force-confold" \
  upgrade
```
`--force-confold` = keep my version; `--force-confdef` = accept the default where there's no conflict. **This exact incantation belongs in every Dockerfile and every provisioning script**, and knowing it cold is a mark of experience.

RPM's approach is different and arguably nicer: it never prompts. It writes `.rpmnew` (the new version, if you'd modified yours) or `.rpmsave` (your version, if it's replacing it) and moves on. Then you go find them:
```bash
$ find /etc -name '*.rpmnew' -o -name '*.rpmsave'
```
Nobody does this, which is why RPM boxes accumulate `.rpmnew` files containing important config changes that never got applied. Both models have a failure mode; RPM's is quieter.

`apt remove` vs `purge` is the same theme: `remove` leaves your config behind (state `rc` in `dpkg -l`), `purge` deletes it. "I removed and reinstalled and it still has my broken config" is `remove` doing exactly what it promised.

### 2.7 Pinning and holding

Stopping an unwanted upgrade:
```bash
sudo apt-mark hold docker-ce docker-ce-cli     # simple, blunt
sudo dnf versionlock add nginx                 # needs python3-dnf-plugin-versionlock
```

APT pinning is more expressive (and more confusing):
```bash
$ cat /etc/apt/preferences.d/nginx
Package: nginx*
Pin: version 1.24.*
Pin-Priority: 1001            # >1000 allows DOWNGRADES too
```
Priorities: `<0` never install; `100` installed-but-not-in-a-repo; `500` default for a normal repo; `990` the target release; `>1000` force, allowing downgrade. `apt-cache policy nginx` shows you the resulting priorities, which is how you debug pinning that "isn't working."

Automatic security updates:
```bash
$ sudo apt install unattended-upgrades
$ cat /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";     # security ONLY — the sane default
};
Unattended-Upgrade::Automatic-Reboot "false";       # think VERY hard before "true"
$ sudo unattended-upgrade --dry-run --debug
```
```bash
$ sudo dnf install dnf-automatic
$ sudo systemctl enable --now dnf-automatic.timer
```

The trade-off is real and worth articulating: **unattended security updates mean unreviewed changes on your production servers**, but the alternative is unpatched CVEs, and the data says unpatched-known-CVE is a far bigger cause of breaches than a bad patch. The standard compromise: auto-apply **security** updates only, never auto-reboot, and monitor for pending reboots (`/var/run/reboot-required`). For truly critical systems, stage it: auto-apply in staging, alert on the diff, apply to prod with a canary.

### 2.8 The other package managers, and choosing between them

| Layer | Examples | Scope | When |
|---|---|---|---|
| **Distro** | `apt`, `dnf`, `apk` | system-wide, curated, signed | **the default choice** |
| **Language** | `pip`, `npm`, `cargo`, `go` | per-project libraries | your app's own dependencies |
| **Container** | Docker images | fully isolated userland | when you need the whole environment pinned |
| **Universal** | Snap, Flatpak, AppImage | sandboxed desktop apps | GUI apps needing newer-than-distro |
| **Functional** | Nix, Guix | reproducible, multi-version | when reproducibility is the requirement |
| **From source** | `./configure && make` | whatever you decide | last resort |

**The rules of thumb worth stating:**
- **System libraries and daemons → distro packages.** They're signed, curated, security-patched, and integrated with systemd/logging/users.
- **Your app's language libraries → the language package manager, in a venv/lockfile.** Never `sudo pip install` — it fights `dpkg` for `/usr/lib/python3/dist-packages` and can break OS tools that depend on a specific version (Ubuntu's `apt` itself is Python). This is why PEP 668 now makes `pip install` outside a venv an error on Debian/Ubuntu, and it's the right call.
- **The mixing problem is real**: `apt install python3-requests` and `pip install requests` install two different `requests` at different paths, and which one you get depends on `sys.path` order. That's a genuine "works on my machine" generator.

**Snap vs Flatpak vs AppImage** — worth having an opinion:
- **Snap** (Canonical): confined, auto-updating, single vendor-controlled store, notoriously slow first launch, and mounts a squashfs per snap (which is why `df` looks insane on Ubuntu desktop). Auto-update is **not disableable**, which is disqualifying for servers.
- **Flatpak**: desktop-focused, sandboxed via bubblewrap + portals, decentralised (Flathub is the main remote but not the only one), better sandboxing story.
- **AppImage**: one file, no install, no sandbox, no updates.
For servers, the answer is essentially always "distro packages or containers"; Snap/Flatpak are desktop technologies.

**Building from source** — the escape hatch:
```bash
$ ./configure --prefix=/usr/local     # /usr/local, NEVER /usr — that's the package manager's
$ make -j$(nproc)
$ sudo make install                   # unmanaged: no database entry, no uninstall, no updates
```
The problem: `make install` scatters files with no record, so there's no `uninstall` and no security updates. Better:
```bash
$ sudo checkinstall                   # builds a .deb from `make install` — now dpkg knows about it
$ sudo apt install ./myapp_1.0_amd64.deb
```
Or use `/usr/local/stow/`, or build a proper package with `fpm`. **The FHS split is the key insight**: `/usr` belongs to the package manager, `/usr/local` belongs to you, and never mixing them is what keeps a system maintainable. That's a Lesson 01 callback and it's the reason the split exists at all.

---

## 3. Official documentation quotes

> "`dpkg` is a tool to install, build, remove and manage Debian packages. The primary and more user-friendly front-end for `dpkg` is `aptitude`. `dpkg` itself is controlled entirely via command line parameters… **`dpkg` maintains some usable information about available packages.**"
> — `man 1 dpkg`, [dpkg(1) — Debian manual](https://manpages.debian.org/stable/dpkg/dpkg.1.en.html)

> "`apt` … provides a high-level commandline interface for the package management system. It is intended as an end user interface and enables some options better suited for interactive usage by default compared to more specialized APT tools like `apt-get` and `apt-cache`."
> — `man 8 apt`, [apt(8) — Debian manual](https://manpages.debian.org/stable/apt/apt.8.en.html)

> "`update` is used to download package information from all configured sources. … **An `update` should always be performed before an `upgrade` or `full-upgrade`.**"
> — `man 8 apt`, [apt(8) — Debian manual](https://manpages.debian.org/stable/apt/apt.8.en.html)

> "This option [`apt-key`] is deprecated and will be removed in a future release. … **Instead of using this command a keyring should be placed directly in the `/etc/apt/trusted.gpg.d/` directory with a descriptive name and either 'gpg' or 'asc' as file extension**, or the `Signed-By` option should be used in the sources.list entry to restrict the key to a specific repository."
> — `man 8 apt-key`, [apt-key(8) — Debian manual](https://manpages.debian.org/stable/apt/apt-key.8.en.html)

> "`Recommends` — This declares a strong, but not absolute, dependency. **The `Recommends` field should list packages that would be found together with this one in all but unusual installations.**"
> — *Debian Policy Manual*, [7.2. Binary Dependencies](https://www.debian.org/doc/debian-policy/ch-relationships.html)

> "Local configuration files … must be preserved during a package upgrade, and … must be preserved when the package is removed (but not purged). … The easy way to achieve this behavior is to make the configuration file a `conffile`."
> — *Debian Policy Manual*, [10.7. Configuration files](https://www.debian.org/doc/debian-policy/ch-files.html#configuration-files)

---

## 4. Real-world examples

### 4.1 Dissect a package before trusting it

```bash
$ apt download nginx-core && ls *.deb
nginx-core_1.24.0-1_amd64.deb

$ ar t nginx-core_1.24.0-1_amd64.deb
debian-binary
control.tar.zst
data.tar.zst

$ dpkg-deb -I nginx-core_1.24.0-1_amd64.deb | head -12    # the control metadata
 Package: nginx-core
 Version: 1.24.0-1
 Depends: libc6 (>= 2.34), libcrypt1 (>= 1:4.1.0), libpcre2-8-0 (>= 10.22), ...
 Provides: httpd, nginx
 Conflicts: nginx-extras, nginx-light

$ dpkg-deb -c nginx-core_1.24.0-1_amd64.deb | wc -l       # how many files?
42
$ dpkg-deb -e nginx-core_1.24.0-1_amd64.deb /tmp/ctl
$ cat /tmp/ctl/postinst                                   # WHAT WILL RUN AS ROOT?
#!/bin/sh
set -e
if [ "$1" = "configure" ]; then
    ...
fi
#DEBHELPER#
```

**Reading `postinst` before installing a third-party package is the correct instinct**, and almost nobody does it. When a vendor hands you a `.deb`, this three-command sequence — `dpkg-deb -I` (what does it claim?), `dpkg-deb -c` (what files?), `dpkg-deb -e` + read the scripts (what runs as root?) — is your entire due diligence, and it takes ninety seconds.

### 4.2 The trust chain, verified by hand

```bash
$ sudo apt update
Get:1 http://archive.ubuntu.com/ubuntu jammy InRelease [270 kB]
Get:2 http://security.ubuntu.com/ubuntu jammy-security InRelease [110 kB]

$ ls /var/lib/apt/lists/*InRelease
/var/lib/apt/lists/archive.ubuntu.com_ubuntu_dists_jammy_InRelease

$ head -20 /var/lib/apt/lists/archive.ubuntu.com_ubuntu_dists_jammy_InRelease
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

Origin: Ubuntu
Suite: jammy
Codename: jammy
Date: Thu, 21 Apr 2022 17:16:08 UTC
...
SHA256:
 8a2e1f...  1234567  main/binary-amd64/Packages.gz    # <-- hash of the package INDEX
```

There's the chain, visible: the `InRelease` file is **signed**, and it contains the **SHA256 of the Packages index**, which in turn contains the **SHA256 of every `.deb`**. Verify the signature and every hash below it follows.

```bash
$ gpg --verify /var/lib/apt/lists/archive.ubuntu.com_ubuntu_dists_jammy_InRelease
gpg: Signature made Thu 21 Apr 2022 17:16:08 UTC
gpg:                using RSA key 871920D1991BC93C
gpg: Good signature from "Ubuntu Archive Automatic Signing Key (2018)"

$ apt-key list 2>/dev/null | head       # deprecated, but shows the legacy global keyring
$ ls /etc/apt/trusted.gpg.d/ /etc/apt/keyrings/
```

Now break it deliberately, to see the failure:
```bash
$ sudo sed -i 's/^Date:.*/Date: Thu, 01 Jan 2030 00:00:00 UTC/' \
    /var/lib/apt/lists/archive.ubuntu.com_ubuntu_dists_jammy_InRelease
$ sudo apt update
E: Release file for http://archive.ubuntu.com/ubuntu/dists/jammy/InRelease is not valid yet
W: GPG error: ... The following signatures were invalid: BADSIG ...
E: The repository is not signed.
```
`apt` refuses. **That refusal is the entire security model working**, and it's worth seeing once. Note that people routinely defeat it with `--allow-unauthenticated` or `[trusted=yes]` in a sources.list line — which is the packaging equivalent of `curl -k`, and equally indefensible.

### 4.3 Dependency archaeology

```bash
$ apt-cache policy nginx
nginx:
  Installed: 1.18.0-6ubuntu14.4
  Candidate: 1.18.0-6ubuntu14.4
  Version table:
 *** 1.18.0-6ubuntu14.4 500
        500 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu jammy-security/main amd64 Packages
        100 /var/lib/dpkg/status                 # <-- 100 = installed, not from any repo
     1.18.0-6ubuntu14 500
        500 http://archive.ubuntu.com/ubuntu jammy/main amd64 Packages
```
That output answers "what's installed, what would install, from where, and at what priority" in one shot — the first command for any "apt is doing something weird."

```bash
$ apt-cache rdepends --installed libpcre2-8-0 | head    # WHO depends on this?
libpcre2-8-0
Reverse Depends:
  nginx-core
  grep
  libglib2.0-0
  ...
```
**`rdepends` is the "can I safely remove this?" tool.** `grep` depends on `libpcre2-8-0`, so removing it takes `grep` with it — and a surprising amount of your system after that.

Watch `autoremove` be dangerous:
```bash
$ sudo apt install nginx        # pulls in nginx-core, nginx-common, libnginx-mod-*
$ sudo apt remove nginx         # removes ONLY the metapackage
$ sudo apt autoremove --dry-run # now the deps are orphaned
The following packages will be REMOVED:
  libnginx-mod-http-geoip2 libnginx-mod-http-image-filter nginx-common nginx-core
```
`autoremove` decides based on the **auto-installed flag** (`apt-mark showauto`). If you manually installed something that's *also* a dependency, and then remove the parent, `autoremove` correctly keeps it. But if a package got auto-installed and you've since come to depend on it directly, `autoremove` will silently take it away. `apt-mark manual <pkg>` is the fix, and "autoremove deleted something I needed" is always this.

### 4.4 The Docker layer-caching trap

```dockerfile
# BAD — three separate problems
FROM ubuntu:22.04
RUN apt-get update                          # cached layer: a STALE index
RUN apt-get install -y nginx curl           # uses the stale index -> "package not found"
                                            # + Recommends bloat + apt cache in the image
```
```dockerfile
# GOOD
FROM ubuntu:22.04
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        nginx=1.18.0-6ubuntu14.4 \
        curl \
    && rm -rf /var/lib/apt/lists/*
```

Four distinct fixes in one line, each worth understanding:
1. **`update` and `install` in the same `RUN`** — otherwise Docker caches the `update` layer, and weeks later `install` runs against an index referencing versions that have been superseded and deleted from the mirror. The error is `E: Version '1.18.0-6ubuntu14.2' for 'nginx' was not found`, and it's baffling until you know.
2. **`--no-install-recommends`** — `Recommends` are installed by default, and they're most of your image bloat. This routinely saves 100–300 MB.
3. **`rm -rf /var/lib/apt/lists/*`** — the index is ~40 MB and useless in the final image. Must be in the *same* `RUN`, or the layer already contains it.
4. **`DEBIAN_FRONTEND=noninteractive`** — otherwise a package with a `debconf` prompt (`tzdata` is the classic) hangs the build forever.

**Version pinning is the reproducibility trade-off**: pin and your build is reproducible but you must actively update to get security fixes; don't pin and you get fixes automatically but your build isn't reproducible and can break without a code change. The mature answer is to pin *and* automate the bumps (Renovate/Dependabot), so you get both — reproducibility plus a PR when a new version lands.

### 4.5 The CVE question

Your security team asks: "are we affected by CVE-2023-38545 (curl SOCKS5 heap overflow)?"

```bash
$ dpkg -l curl libcurl4
ii  curl      7.81.0-1ubuntu1.14   amd64
ii  libcurl4  7.81.0-1ubuntu1.14   amd64
```
Upstream says "fixed in 7.71.0"... no wait, "affects 7.69.0 to 8.3.0, fixed in **8.4.0**." You have 7.81.0. **Are you vulnerable?**

The naive answer is yes. The correct answer requires understanding **backporting**:
```bash
$ apt changelog curl | head -20
curl (7.81.0-1ubuntu1.14) jammy-security; urgency=medium
  * SECURITY UPDATE: heap buffer overflow in SOCKS5 proxy handshake
    - debian/patches/CVE-2023-38545.patch: fix in lib/socks.c
    - CVE-2023-38545
 -- Ubuntu Security <security@ubuntu.com>  Wed, 11 Oct 2023 09:00:00 +0000
```
**Fixed.** The version is still 7.81.0 because Ubuntu **backports** the patch rather than upgrading — that's the entire value of a stable distro: security fixes without behaviour changes. So **you cannot determine vulnerability from the upstream version number alone**, and every naive CVE scanner that does version-range matching against a distro package produces false positives. This is a genuinely important real-world point that catches out a lot of people, and it's why distro security trackers exist:

```bash
$ dnf updateinfo list --security                    # RHEL: pending security updates
$ dnf updateinfo info CVE-2023-38545
$ apt list --upgradable | grep -i security
# Authoritative sources:
#   https://ubuntu.com/security/CVE-2023-38545
#   https://security-tracker.debian.org/tracker/CVE-2023-38545
#   https://access.redhat.com/security/cve/CVE-2023-38545
```

Being able to explain "the version says 7.81.0 but it's patched, here's the changelog, here's the distro's CVE tracker entry" is a genuinely senior response to a security ticket.

---

## 5. How This Is Used In Production

- **Startups:** `apt install` on a VM, and increasingly not even that — a Dockerfile is the whole story. The failure mode is unpinned everything, so a rebuild six months later produces a different image and something subtly breaks. The second failure is `curl | sudo bash` from a vendor README, which is unauditable, unversioned, unremovable, and gets no security updates.
- **Enterprises:** **internal mirrors** are mandatory — Artifactory, Nexus, Pulp, or `apt-mirror`. Reasons: bandwidth and speed; **availability** (upstream mirrors go down and you can't deploy); **control** (only approved packages, scanned and staged); **auditability** (what did we install, when, from where); and **air-gapped** environments where there's no upstream at all. Packages get promoted through environments like code: dev → staging → prod, with a soak period. Custom internal software is packaged as `.deb`/`.rpm` (via `fpm` or native tooling) so it uses the same distribution and rollback machinery as everything else — which is why "just scp the binary" is a hard no.
- **Common architectures:** **immutable infrastructure** — don't patch servers, rebuild images and replace them (Packer bakes an AMI, or you rebuild the container). This makes package management a **build-time** concern with a reproducible artifact, and it eliminates configuration drift entirely, which is the real win. The alternative — **config management** (Ansible/Puppet declaring package state) — converges mutable servers and is still what you'll find in most enterprises. And **SBOM** (Software Bill of Materials) generation from the package database (`syft`, `trivy`) is now a compliance requirement in many sectors, driven by executive orders after SolarWinds and Log4Shell.
- **Scaling:** a fleet of 5,000 servers all hitting `archive.ubuntu.com` at 03:00 is a self-inflicted DDoS and gets you rate-limited; you need a mirror and jittered schedules. Rolling patch windows with canaries: patch 1%, watch, then 10%, then the rest. `unattended-upgrades` for security patches on the long tail, with **no auto-reboot** and monitoring on `/var/run/reboot-required`. Kernel updates need a reboot (or **livepatch** — Canonical's Livepatch, `kpatch`, `kexec`) which is why "patched" and "running the patched code" are different questions, and why `needrestart` exists.
- **Monitoring & logging:** alert on — pending security updates older than N days; `/var/run/reboot-required` present for >7 days; **unsigned or `gpgcheck=0` repos** in config; unexpected repos added; `dpkg -V`/`rpm -Va` integrity failures; and disk filling from `/var/cache/apt/archives` (a real cause of full disks). `/var/log/apt/history.log` and `dnf history` are your change log — correlate an incident's start time against them first, because "what changed?" is answered there surprisingly often.
- **Security:** this is **supply chain**, and it's the highest-stakes topic in the lesson. **Every install runs maintainer scripts as root**, so trust is absolute. The **`apt-key`/`signed-by` story** is the concrete lesson: an unscoped key could sign for any repo, so one bad PPA compromised your whole base system. **Never `gpgcheck=0`** or `[trusted=yes]`. **Never `curl | sudo bash`** (§11 Q4). Real incidents worth knowing: **event-stream** (npm, 2018 — a maintainer handed over a package to an attacker who added a crypto stealer); **XZ Utils / CVE-2024-3094** (2024 — a multi-year social-engineering campaign put an SSH backdoor into a core compression library, caught by a Postgres engineer noticing a 500 ms latency anomaly, which should terrify you); **SolarWinds** (2020 — build-system compromise). Defences: signed packages, reproducible builds, SBOMs, vendoring with lockfiles, and minimising your dependency surface.
- **CI/CD:** pin versions for reproducibility, automate the bumps (Renovate/Dependabot) so you get security fixes anyway. Cache package downloads to speed builds. Scan images (`trivy`, `grype`) — but **understand backporting**, or you'll drown in false positives from version-range matching (§4.5). Generate an SBOM per build and store it, so that when the next Log4Shell lands you can answer "where are we affected?" in minutes instead of weeks. That last point is the actual business case for SBOMs and worth saying out loud.
- **Common tools:** `apt`/`dpkg`, `dnf`/`rpm`, `apk`, `apt-cache policy`, `debsums`, `fpm` (build a package from anything), `checkinstall`, `equivs`, `reprepro`/`aptly`/`createrepo` (run your own repo), Artifactory/Nexus/Pulp, `unattended-upgrades`/`dnf-automatic`, `needrestart`, `trivy`/`grype`/`syft`, Packer, Nix.
- **Real examples:** **Debian**'s reproducible-builds project is the reference effort for verifying that a binary genuinely corresponds to its source — a direct response to the trust problem in §2.2. **Google** and **Meta** run enormous internal mirrors and custom packaging because upstream availability and consistency at their scale is non-negotiable. **Netflix**'s immutable-AMI model (bake with Packer, never patch in place) is the canonical implementation of the immutable-infrastructure argument. And **XZ/CVE-2024-3094** is the case study every platform engineer should be able to discuss: it demonstrates that the trust chain is only as strong as the *humans* in it — the signatures were all valid, because the attacker had become a legitimate maintainer.

---

## 6. Advanced corner

**Dependency resolution is NP-complete.** "Find a set of package versions satisfying all constraints" is Boolean satisfiability. Real solvers use heuristics; modern ones (`libsolv`, used by `dnf` and `zypper`) literally use a **SAT solver**. This is why `apt` sometimes proposes a bizarre solution, why it can take seconds on a complex transaction, and why `apt` and `aptitude` can disagree about how to fix the same conflict — different heuristics, both valid. It's also why **lockfiles** exist in language ecosystems: solve once, record the answer, replay it deterministically. Understanding that dependency resolution is a search problem — not a lookup — reframes a lot of frustration.

**Version comparison is not string comparison and not numeric.** Debian's algorithm handles epochs, upstream versions, and revisions: `1:1.2.3-4ubuntu1` = epoch `1`, upstream `1.2.3`, revision `4ubuntu1`. The **epoch** exists to fix mistakes — if upstream re-versions from `20230101` to `1.0`, `1.0 < 20230101` by any normal comparison, so you bump the epoch and `1:1.0 > 0:20230101` forever. Epochs are sticky and never go away, which is why maintainers hate them.
```bash
$ dpkg --compare-versions 1.0 lt 1.0.1 && echo yes        # yes
$ dpkg --compare-versions 1.0~rc1 lt 1.0 && echo yes      # yes — '~' sorts BEFORE everything
$ dpkg --compare-versions 1.0 lt 1.0a && echo yes         # yes
$ rpmdev-vercmp 1.0 1.0.1
```
**The `~` is the beautiful part**: it sorts before the empty string, so `1.0~beta1 < 1.0~rc1 < 1.0`. That's how pre-releases are expressed without epochs. Nobody knows this and it's genuinely useful.

**Backporting explains the "vulnerable version" false positive.** Stable distros freeze upstream versions and **backport** security patches, so `curl 7.81.0-1ubuntu1.14` contains fixes that upstream only shipped in 8.4.0. Consequences: (1) version-range CVE scanners produce mountains of false positives against distro packages — you must use the distro's OVAL/security-tracker data, which is exactly what `trivy` does when it recognises the distro and what naive scanners don't; (2) `--version` output tells you nothing about patch level; (3) `apt changelog` / `rpm -q --changelog` is the authoritative local answer. This single concept resolves the most common friction between security and platform teams, and being able to explain it calmly is disproportionately valuable.

**Debian's `Recommends` is installed by default and RPM's `Recommends` is a "weak dependency" — same idea, different history.** The `--no-install-recommends` / `--setopt=install_weak_deps=False` flags are the same lever. The design tension is real: `Recommends` makes desktop installs work out of the box and makes container images enormous. Debian's own policy says `Recommends` should be things you'd find together "in all but unusual installations" — and a container **is** an unusual installation, which is precisely the argument for always disabling them there.

**Multi-arch is why `libc6:i386` exists.** Debian's multi-arch support lets packages from different architectures coexist, with libraries in `/usr/lib/x86_64-linux-gnu/` rather than `/usr/lib/`. That path scheme — which looks like pointless verbosity — is what makes it possible to have both amd64 and i386 libraries installed simultaneously (for Steam, Wine, or 32-bit legacy binaries). `dpkg --add-architecture i386` enables it. It's a nice example of a design constraint producing a visible convention.

**`apt` vs `apt-get`.** `apt` is the newer, human-facing front-end: progress bars, colour, sensible defaults, and — critically — **an unstable CLI that may change between releases**. `apt-get` has a **stable, guaranteed** interface. So the rule is: **`apt` interactively, `apt-get` in scripts.** `apt` even warns you: *"WARNING: apt does not have a stable CLI interface. Use with caution in scripts."* People ignore this and then their CI breaks on a distro upgrade.

**Alpine and musl.** Alpine uses `apk` and **musl libc** instead of glibc, which makes images tiny (5 MB base) but breaks anything expecting glibc — most notably **Python wheels**, which are built against glibc (`manylinux`), so `pip install` on Alpine falls back to compiling from source. That's why an Alpine Python image can *build slower and end up larger* than the Debian slim one, which is a genuinely counter-intuitive result that trips up a lot of teams optimising image size. There are also real DNS resolution differences (musl's resolver doesn't handle some search-domain edge cases the same way) that produce baffling Kubernetes bugs. **`python:3.12-slim` is usually the right default, not `python:3.12-alpine`** — a specific, useful, contrarian recommendation.

**Nix inverts the whole model.** Instead of `/usr/lib/libfoo.so` (one version, globally), Nix installs to `/nix/store/<hash>-libfoo-1.2.3/`, where the hash covers **every input** to the build. So multiple versions coexist trivially, installs are atomic, rollback is instant (repoint a symlink), and builds are reproducible by construction. It solves dependency hell definitively — at the cost of a completely different mental model, a language to learn, and an ecosystem that fights the FHS. It's the most intellectually satisfying answer in the space and the hardest sell to a team.

**What most people get wrong at this level:**
- `apt update` vs `upgrade`.
- Assuming a version number tells you the patch level (backporting).
- `sudo pip install` alongside distro Python packages.
- `apt update` in a separate Docker layer from `install`.
- Believing HTTPS is what makes packages trustworthy (signing is).
- Not knowing `Recommends` are installed by default.
- Using `apt` in scripts.
- Choosing Alpine reflexively for "smaller images."
- `make install` into `/usr` instead of `/usr/local`.

---

## 7. Best practices

- **Distro packages first.** They're signed, curated, patched, and integrated. Reach past them only with a reason you can articulate.
- **Never `curl | sudo bash`.** Download, read, then run — or better, use a signed repository. If a vendor offers only a pipe-to-shell, that's information about the vendor.
- **`signed-by=` for every third-party repo.** Never `apt-key add`, never `gpgcheck=0`, never `[trusted=yes]`.
- **`apt-get` in scripts, `apt` interactively.** `apt`'s CLI is explicitly unstable.
- **In Dockerfiles:** `update && install` in one `RUN`, `--no-install-recommends`, `rm -rf /var/lib/apt/lists/*`, `DEBIAN_FRONTEND=noninteractive`. Pin versions *and* automate the bumps.
- **Never `sudo pip install`.** Use a venv (and note PEP 668 now enforces this on Debian/Ubuntu). Same for `npm -g` on a system Node.
- **`/usr/local` for anything you build.** `/usr` belongs to the package manager. Better still, package it with `fpm`/`checkinstall` so it can be removed and updated.
- **Automate security updates** (`unattended-upgrades` restricted to `-security`), never auto-reboot, and monitor `/var/run/reboot-required`.
- **`apt-mark hold` / `dnf versionlock`** for anything where a surprise upgrade would be an outage (database engines, kernels on tuned hosts) — and document *why* next to the hold, because an undocumented hold becomes an unpatched CVE.
- **Use `apt-cache policy` first** when apt does something confusing. It answers most questions in one shot.
- **Read the maintainer scripts** (`dpkg-deb -e` / `rpm -q --scripts`) before installing anything from a vendor.
- **Use the distro's CVE tracker**, not upstream version ranges, to assess vulnerability.
- **Run an internal mirror** at any real scale — for availability as much as bandwidth.
- **`--dry-run` / `-s`** before any bulk operation, and **read what `autoremove` proposes** before agreeing.

## 8. Common mistakes & gotchas

**`apt update` ≠ `apt upgrade`** — `update` refreshes the index and changes nothing.

**Docker layer caching a stale index**
```dockerfile
RUN apt-get update              # cached forever
RUN apt-get install -y nginx    # E: Version '...' was not found
```

**`sudo pip install` fighting `dpkg`** — two `requests` at different paths, and `sys.path` order decides. Can break `apt` itself (it's Python). PEP 668 now blocks this by default.

**Assuming the version number tells you the patch level** — backporting means `7.81.0` may contain an `8.4.0` fix.

**`remove` vs `purge`** — `remove` leaves config (state `rc`). "I reinstalled and it kept my broken config" is always this.

**`autoremove` taking something you needed** — it goes by the auto-installed flag. `apt-mark manual <pkg>` to protect it.

**Conffile prompts hanging automation** — always `DEBIAN_FRONTEND=noninteractive` + `--force-confold`.

**`.rpmnew` files accumulating** — RPM never prompts, it just writes the new config alongside and moves on. Nobody looks. `find /etc -name '*.rpmnew'`.

**`apt` in a script breaking on a distro upgrade** — unstable CLI, and it warns you.

**Mixing distro and language packages for the same library** — `apt install python3-requests` + `pip install requests` = two installs, undefined winner.

**`make install` into `/usr`** — collides with the package manager, and `dpkg` will happily overwrite your files on the next upgrade.

**Full `/var/cache/apt/archives`** — a genuine cause of full-disk incidents. `apt clean`.

**Alpine + Python** — musl means no `manylinux` wheels, so pip compiles from source. Often slower and bigger than `-slim`.

## 9. Where AI helps (and where it hurts)

- **Helps:** remembering the exact `dpkg-query`/`rpm -q` incantation for "which package owns this file"; writing the correct modern `signed-by` repo setup (it does know the pattern); explaining a dependency conflict message; generating the noninteractive-upgrade flag soup.
- **Hurts:** AI confidently emits **`apt-key add`**, because that's what a decade of blog posts say, and it's deprecated *for a security reason* it won't mention. It writes **`apt update` and `apt install` in separate Dockerfile `RUN` layers**. It suggests `sudo pip install`. It recommends `curl | sudo bash` because that's what the vendor's README says. And it will tell you a package is vulnerable based on the **upstream version number**, missing backporting entirely — which will send you chasing a non-existent CVE. **Check any repo-setup instructions against the vendor's current official docs**, because this is an area where the training data is systematically out of date and the errors are security-relevant.

## 10. Learn independently

**Books (specific chapters):**
- *UNIX and Linux System Administration Handbook*, 5th ed. (Nemeth et al.) — **Ch. 6 "Software Installation and Management"** — the best single chapter on this topic anywhere: covers both families side by side, repositories, and the operational reality of fleet patching. This is the primary reference for this lesson.
- *How Linux Works*, 3rd ed. (Ward) — **Ch. 16 "Introduction to Compiling Software From C Source Code"** and **Ch. 17 "Virtualization"** — the `./configure && make && make install` chapter is exactly the "when you must build from source" material, and Ward explains *why* `/usr/local` matters.
- *The Linux Command Line* (Shotts) — **Ch. 14 "Package Management"** — a clean, gentle overview of the two-layer model with the command tables. Free at [linuxcommand.org](https://linuxcommand.org/tlcl.php).
- *The DevOps Handbook* (Kim, Humble, Debois, Willis) — **Part IV, "The Second Way"** and the chapters on **deployment pipelines and immutable infrastructure**. The argument for "rebuild, don't patch" and for treating packages as versioned artifacts promoted through environments — the conceptual backing for §5's architecture section.
- *Site Reliability Engineering* (Google, free at [sre.google/books](https://sre.google/books/)) — **Ch. 8 "Release Engineering"** — hermetic builds, reproducibility, and why Google packages everything. Directly relevant to the pinning/reproducibility trade-off.
- *The Debian Administrator's Handbook* (Hertzog & Mas) — **Ch. 5 "Packaging System"** and **Ch. 6 "Maintenance and Updates: The APT Tools"** — free at [debian-handbook.info](https://debian-handbook.info/), and the most thorough Debian-specific treatment in existence.

**Primary docs:**
- [**Debian Policy Manual**](https://www.debian.org/doc/debian-policy/) — **Ch. 7 "Declaring relationships between packages"** (the `Depends`/`Recommends`/`Provides` semantics, normatively) and **§10.7 "Configuration files"** (the conffile rules). This is the specification.
- [`man 1 dpkg`](https://manpages.debian.org/stable/dpkg/dpkg.1.en.html), [`man 8 apt`](https://manpages.debian.org/stable/apt/apt.8.en.html), [`man 5 apt_preferences`](https://manpages.debian.org/stable/apt/apt_preferences.5.en.html) — the last one is the pinning reference and it's genuinely hard to understand from anywhere else.
- [`man 5 sources.list`](https://manpages.debian.org/stable/apt/sources.list.5.en.html) — including the `signed-by` option and the deb822 format.
- [DNF documentation](https://dnf.readthedocs.io/) and [RPM Packaging Guide](https://rpm-packaging-guide.github.io/) — the latter is excellent and free.
- [Debian New Maintainers' Guide](https://www.debian.org/doc/manuals/maint-guide/) — build your first `.deb` properly.

**High-quality links:**
- [Ubuntu security notices](https://ubuntu.com/security/notices) / [Debian security tracker](https://security-tracker.debian.org/) / [Red Hat CVE database](https://access.redhat.com/security/security-updates/cve) — **the authoritative answer to "are we vulnerable?"**, and the antidote to version-range false positives.
- [Reproducible Builds](https://reproducible-builds.org/) — the project, the rationale, and the current state. Read the "why" page.
- [The XZ backdoor (CVE-2024-3094) — Evan Boehs' timeline](https://boehs.org/node/everything-i-know-about-the-xz-backdoor) — read this end to end. It is the most important supply-chain story of the decade and it's a genuinely gripping piece of writing.
- [PEP 668](https://peps.python.org/pep-0668/) — why `pip install` outside a venv is now an error, told from the packaging-conflict angle.
- [Nix pills](https://nixos.org/guides/nix-pills/) — if you want to see the functional alternative properly.

## 11. Interview preparation

**Q1. `apt` vs `dpkg`?**
> Two layers. `dpkg` is low-level: it installs a single `.deb` file, runs the maintainer scripts, and records what happened in the package database. It **checks** dependencies but **can't resolve** them — hand it a package needing `libpq5` and it errors out. `apt` is high-level: it knows about repositories, computes the transitive dependency graph, downloads what's needed, orders the operations, and then calls `dpkg` to do the actual installing. So `apt` is a dependency solver and downloader wrapped around `dpkg`. The same split is `rpm` and `dnf` on the Red Hat side. You see it directly when `dpkg -i` fails on a missing dependency and `apt install -f` fixes it. A related detail worth adding: use **`apt-get` in scripts** — `apt`'s CLI is explicitly documented as unstable and it warns you about exactly this.

**Q2. How does your system know a package is authentic?**
> A signature chain. The repository publishes an index (`Release`/`InRelease` for Debian, `repomd.xml` for RPM) that lists every package with its SHA256 hash. That index is **GPG-signed** by the repository's key. Your machine holds the corresponding public key and verifies the signature; once the index is verified, every package's integrity follows from the hash in it. **So the whole model rests on the key** — everything else is a hash chain hanging off it. Crucially, **HTTPS is not the security control**: it hides what you're downloading and prevents casual tampering, but the GPG signature is what proves authenticity, which is why Debian mirrors worked fine over plain HTTP through caching proxies for years. You can see the whole thing by hand: `gpg --verify` the `InRelease` file in `/var/lib/apt/lists/` and read the SHA256 section inside it.

**Q3. Why was `apt-key add` deprecated?**
> Because it put keys into a **global trusted keyring**, so any key could sign for **any** repository. Adding some random PPA's key gave that key authority to sign a replacement `openssh-server` or `libc6` that appeared to come from the official Ubuntu archive — one sketchy third-party repo silently compromised your entire base system. The replacement is `signed-by=/etc/apt/keyrings/vendor.gpg` in the sources.list entry, which **scopes the key to exactly one repository** — which is what everyone assumed `apt-key` was doing all along. It's a good example of a security model where the mental model and the implementation diverged for years. The practical consequence: any tutorial telling you to `apt-key add` is out of date, and AI tools reproduce it constantly because the training data is full of it.

**Q4. Why not `curl https://example.com/install.sh | sudo bash`?**
> Several reasons, and they compound. **You can't read what you're running** — and the server can serve different content to `curl` than to a browser, so "I checked it in my browser" proves nothing; it can even detect a slow pipe and change behaviour mid-stream. **It executes as root** with no review. **There's no version, no integrity check, no signature** — you're trusting TLS and DNS, not the publisher. **There's no uninstall** — files are scattered with no database record. **There are no security updates** — it's outside your patch process forever, so it's invisible to your CVE tooling and your SBOM. And **it usually adds a repo and a key anyway**, so it's an unauditable wrapper around the thing you should have done deliberately. The better path: download it, read it, then run it — or use the vendor's signed repository, which most of them have. If a vendor offers *only* pipe-to-shell, that tells you something about how they think about their users' security. That said, I'd acknowledge the pragmatic counter: the script is often served from the same TLS-protected origin as the repo and key you'd otherwise trust, so the marginal risk is smaller than the rhetoric suggests — the real objections are auditability, removability, and patchability, not just "someone could MITM you."

**Q5. What's the difference between `apt remove` and `apt purge`?**
> `remove` deletes the package's binaries but **keeps its configuration files** — you'll see the package in state `rc` in `dpkg -l` ("removed, config-files remain"). `purge` removes the config too. That's why "I removed it and reinstalled and it still has my broken config" happens — `remove` did exactly what it promised. It's deliberate: it means you can remove a package temporarily without losing your carefully-tuned `/etc` files. `autoremove` is the third one, and it's separate: it drops packages that were installed *automatically* as dependencies and are now orphaned, based on the auto-installed flag — which is why it occasionally removes something you'd come to depend on directly, and `apt-mark manual` is the fix.

**Q6. What's dependency hell, and why is it hard?**
> The situation where package A needs `libfoo >= 2.0`, package B needs `libfoo < 2.0`, and both must be installed. It's genuinely hard because the general problem — "find a set of package versions satisfying all constraints" — is **Boolean satisfiability, which is NP-complete**. Modern resolvers (`libsolv` in `dnf` and `zypper`) literally use SAT solvers. That's why `apt` sometimes proposes a surprising solution, why complex transactions take seconds, and why `apt` and `aptitude` can disagree about how to fix the same conflict — different heuristics, both valid. The mitigations are architectural: **virtual packages** (`Provides: httpd` lets nginx or apache satisfy the same dependency), **versioned sonames** (`libfoo.so.1` and `libfoo.so.2` coexisting), **containers** (give each app its own userland), and **Nix**, which sidesteps it entirely by content-addressing every build so multiple versions coexist by construction. Language ecosystems solve it with **lockfiles** — solve once, record the answer, replay deterministically.

**Q7. (System design) Design package management for 500 servers across dev/staging/prod.**
> The goal is **reproducible, auditable, available, and patched** — in that order of difficulty. **Internal mirror** first (Artifactory/Nexus/Pulp): upstream goes down and you can't deploy, which is an availability incident caused by someone else's infrastructure; a mirror also gives bandwidth, control over what's approved, and an audit trail. **Promote packages like code** — the same versioned artifact moves dev → staging → prod with a soak period; never rebuild per environment, because then you're not testing what you ship. **Prefer immutable infrastructure**: bake images with Packer or build containers, and replace servers rather than patching them — this eliminates configuration drift, which is the failure mode that actually bites at 500 servers, where no two boxes are quite the same after two years of `apt install`. If mutable servers are unavoidable, config management (Ansible/Puppet) declares package state and converges. **Patching:** `unattended-upgrades` restricted to `-security` on the long tail, never auto-reboot, monitor `/var/run/reboot-required`; for anything critical, rolling windows with canaries — 1%, watch, 10%, rest. **Pinning** on anything where a surprise upgrade is an outage (database engines, kernels on tuned hosts), documented next to the hold, with Renovate raising PRs so a pin doesn't silently become an unpatched CVE. **SBOM per build** (`syft`), stored — because the real payoff is answering "where are we affected?" in minutes when the next Log4Shell lands, instead of spending three weeks grepping. And **jitter the schedules** — 500 boxes hitting the mirror simultaneously at 03:00 is a self-inflicted DDoS. The thing I'd push back on if someone proposed it: skipping the mirror to "keep it simple." That's the piece that turns someone else's outage into your outage.

**Q8. (Trade-off) Should you pin package versions in a Dockerfile?**
> It's a genuine trade-off, and the honest answer is "pin *and* automate." **Pinning** gives reproducibility — the same Dockerfile produces the same image in six months, which is what makes rollbacks meaningful and "works on my machine" debuggable. The cost is that you **stop getting security fixes** until someone bumps the pin, and pins rot silently — you find out at the audit. **Not pinning** gives you fixes automatically, but your build isn't reproducible, it can break with no code change (upstream removed the version, or a new one has a breaking change), and a rebuild of "the same commit" is a different image. The mature answer is both: pin explicitly, and run Renovate/Dependabot to raise a PR when a new version lands — so you get reproducibility *and* patching, with a human in the loop and a CI run per bump. There's also a middle position worth mentioning: pin the **base image by digest** (which is the big reproducibility lever) and let packages float within it, which is a reasonable trade when your base image is rebuilt frequently anyway. What I wouldn't do is pin and *not* automate, because that's how you end up shipping a two-year-old OpenSSL.

**Q9. A CVE says version 8.4.0 fixes it. We have 7.81.0. Are we vulnerable?**
> Probably not, and this is the single most common false positive in security tooling. Stable distros **backport** security patches rather than upgrading versions — that's the entire value proposition of a stable distro: you get the fix without the behaviour change. So Ubuntu's `curl 7.81.0-1ubuntu1.14` contains the patch that upstream shipped in 8.4.0, and the version number deliberately doesn't reflect it. The authoritative checks are `apt changelog curl` / `rpm -q --changelog curl` (look for the CVE ID) and the distro's security tracker — Ubuntu's CVE pages, Debian's security-tracker, Red Hat's CVE database. Any scanner doing naive version-range matching against distro packages will produce mountains of false positives; good ones (`trivy`) use the distro's OVAL data instead. This is worth explaining calmly to a security team, because it's the most common source of friction between security and platform, and being the person who can resolve it with a changelog and a tracker link is disproportionately valuable.

**Q10. Alpine or Debian slim for a Python image?**
> Usually **Debian slim**, which surprises people who assume smaller base = smaller image. Alpine uses **musl libc** instead of glibc, and Python wheels on PyPI are built against glibc (the `manylinux` standard) — so `pip install` on Alpine can't use the prebuilt wheel and **compiles from source** instead. That means you need a compiler toolchain in the image, builds take minutes instead of seconds, and the result is often *larger* than the `-slim` image you were trying to beat. There are also real musl differences that cause subtle production bugs — DNS resolution behaviour differs in ways that bite in Kubernetes, and there have been performance differences in `malloc`-heavy workloads. Alpine is genuinely great when your dependencies are pure Python or you're shipping a static Go binary. For a typical Python service with `numpy`/`psycopg2`/`cryptography` in the tree, `python:3.12-slim` is the right default. It's a good example of an optimisation that's widely cargo-culted from a metric (base image size) that isn't the one you actually care about (build time, final size, and correctness).

**Q11. What happens if you modify `/etc/nginx/nginx.conf` and then upgrade nginx?**
> `nginx.conf` is a **conffile**, so `dpkg` does a three-way comparison: the originally-shipped file, your current file, and the new shipped file. If you didn't modify it, it's silently replaced. If you modified it but the package didn't change it, yours is kept. **If both changed, you get an interactive prompt** offering to keep yours, take theirs, or show a diff. That prompt is the operational trap: **in automation it blocks forever**, which is why every provisioning script and Dockerfile needs `DEBIAN_FRONTEND=noninteractive` plus `-o Dpkg::Options::="--force-confold"` (keep mine) and `--force-confdef` (take the default where there's no conflict). RPM handles it differently — it never prompts, it just writes a `.rpmnew` (or `.rpmsave`) alongside and moves on, which doesn't hang automation but means boxes silently accumulate unapplied config changes that nobody ever looks at. Both models have a failure mode; RPM's is quieter, which is arguably worse.

**Q12. (Senior) How would you evaluate whether to trust a third-party `.deb` a vendor gave you?**
> Ninety seconds of inspection answers most of it. **`dpkg-deb -I`** — what does it claim? Look at `Depends` (is it pulling in something alarming?), `Conflicts`/`Replaces` (is it taking over files from a package I care about?), and the maintainer. **`dpkg-deb -c`** — what files, and where? Anything outside `/opt` or `/usr/local` for a vendor package deserves a question; anything in `/etc/sudoers.d/`, `/etc/cron.d/`, or a setuid bit is a stop-and-read. **`dpkg-deb -e` and read `preinst`/`postinst`/`prerm`/`postrm`** — this is the crux, because **these run as root**, and this is where a package can do literally anything. Then the meta-questions: **is it signed**, and by a key I can verify out-of-band? Is there a **repository** instead, so I get updates and revocation? What's the vendor's **security response** story — will I hear about a CVE in this? Can I **rebuild it from source** if I need to? And structurally: can I avoid the question entirely by running it in a container with dropped capabilities and a non-root user, so a malicious `postinst` compromises a container rather than a host? The honest framing is that inspection catches accidents and lazy malice, not a determined attacker — **XZ/CVE-2024-3094** had valid signatures from a legitimate maintainer, and no amount of `dpkg-deb -e` would have caught it. So the real answer is defence in depth: inspect, isolate, minimise what you install at all, and keep an SBOM so that when trust is broken retroactively you can find out where you're exposed in minutes.

## 12. Homework

> Work in `linux/solutions/lesson-06/`. Use a throwaway VM or container — several tasks install and remove real packages. Do the Debian tasks on Ubuntu/Debian and, where marked, repeat on Fedora/Rocky in a container to compare.

**Easy**

1. **Query drill.** Answer with commands only (no web searching): which package owns `/usr/bin/ssh`? What files does `openssh-client` install? What depends on `libssl3`? What versions of `nginx` are available and from which repos? Has anyone modified any file in `coreutils`? When was `curl` last upgraded and why?
   *Acceptance:* a `QUERIES.md` with the command and output for each; the last two need `dpkg -V`/`rpm -V` and `apt changelog` / `/var/log/apt/history.log`.

2. **Dissect a package.** Download a `.deb` without installing it. Extract and examine: the three `ar` members, the `control` metadata, the full file list, and **every maintainer script**. Write one paragraph on what the `postinst` does and what it could do.
   *Acceptance:* the transcript plus the paragraph; you must correctly identify at least one thing the script does as root.

3. **Two layers, demonstrated.** Find a package with dependencies. Try to install it with `dpkg -i` alone and capture the failure. Fix it with `apt install -f`. Then do it properly with `apt install ./file.deb`. Explain the three behaviours.
   *Acceptance:* the transcript with all three, and a written explanation of the layer boundary.

**Medium**

4. **Verify the trust chain by hand.** Find your `InRelease` file, `gpg --verify` it, and identify the signing key. Extract the SHA256 of the `Packages` index from it and verify that the on-disk index matches. Then **break it**: tamper with the `InRelease` file and show `apt update` refusing. Restore.
   *Acceptance:* the full verified chain (signature → index hash → actual index), the tampering failure with the exact error, and a diagram (ASCII is fine) of the chain.

5. **Add a repo properly.** Add Docker's (or any third-party) repository using the **modern** `signed-by` pattern, from scratch, without copying a one-liner. Then demonstrate the **danger of the old way**: add a key with `apt-key add` (or drop it in `trusted.gpg.d/`) and explain — with `apt-key list` output — why that key could now sign for the *Ubuntu* archive.
   *Acceptance:* a working `signed-by` setup verified with `apt-cache policy docker-ce`; a written explanation of the scoping difference with evidence from the keyring contents.

6. **Dockerfile shootout.** Build four images installing nginx + curl + a Python package: (a) the naive version with separate `RUN apt-get update` / `install` and no cleanup; (b) with `--no-install-recommends`; (c) also with `rm -rf /var/lib/apt/lists/*`; (d) `python:3.12-slim` vs `python:3.12-alpine` for the same Python deps. Measure image size and build time for each.
   *Acceptance:* a results table (MB, build seconds) for all four; a demonstration of the stale-index failure by building (a), waiting for a package update (or simulating it), and rebuilding; and a paragraph on why Alpine was or wasn't smaller — with the actual numbers.

7. **Conffile conflict.** Install a package, modify one of its conffiles, then force an upgrade that also changes that file (use two versions from the archive, or `dpkg -i` an older then newer version). Capture the interactive prompt. Then reproduce it in a script and show it **hanging**. Fix it with the noninteractive flags. Finally, do the RPM equivalent in a Fedora container and find the `.rpmnew`.
   *Acceptance:* the prompt captured, evidence of the hang (with a timeout), the working noninteractive version, and the `.rpmnew` demonstration with a note on which failure mode you consider worse.

**Hard**

8. **Build a real package.** Take a small program (a shell script + a systemd unit + a config file is fine) and package it three ways: (a) `checkinstall` from `make install`; (b) `fpm`; (c) a proper native `.deb` with a `debian/` directory and `dpkg-buildpackage`. Each must install a system user, place config in `/etc/`, install a systemd unit, and **remove cleanly**.
   *Acceptance:* three installable packages; `dpkg -L` showing correct paths for each; clean install → verify → remove → purge cycles; and a comparison of the three approaches (effort, correctness, maintainability) with a recommendation.

9. **Run your own repository.** Set up a signed APT repository with `aptly` or `reprepro`: generate a GPG key, add your package from task 8, publish it, serve it over HTTP, and consume it from a second machine/container using `signed-by`. Then prove the security model: **tamper with a package in the repo** and show the client rejecting it.
   *Acceptance:* a working end-to-end install from your repo on a clean container; the tampering rejection with the exact error; and a `README` explaining the key management (where does the private key live? what happens when it expires? how would you rotate it?).

10. **The CVE investigation.** Pick a real CVE affecting a package on your system (`curl`, `openssl`, `sudo` — check the Ubuntu/Debian security tracker for a recent one). Determine authoritatively whether you're vulnerable, using: the installed version, `apt changelog`, the distro security tracker, and the actual patch in the source. Then run `trivy` on a container image and **find at least one false positive** caused by backporting.
    *Acceptance:* a written assessment for the CVE with evidence at each step; the `trivy` false positive identified and explained; and a paragraph you could send to a security team explaining backporting. This is the most job-relevant task in the lesson.

**Stretch (senior)**

11. **Fleet patching simulation.** With 5 containers as your "fleet", build a patching pipeline: a script that reports pending security updates per host; applies them with proper noninteractive flags; detects whether a reboot is required (`/var/run/reboot-required`) and which services need restarting (`needrestart`); does it as a **canary rollout** (1 host, verify, then the rest); and can **roll back** (on Fedora use `dnf history undo`; on Debian, work out what you'd actually do — this is the interesting part).
    *Acceptance:* the working pipeline; a demonstrated canary that halts on a simulated failure; an honest write-up of the Debian rollback problem and three options (cache the old `.deb`s, snapshot the filesystem, rebuild the image) with a recommendation. The gap between `dnf history undo` and apt's lack of it is the lesson.

12. **Dependency resolution is SAT.** Construct a genuine dependency conflict with real packages (or build synthetic ones with `equivs`): A needs `libfoo >= 2`, B needs `libfoo < 2`, and both are required by C. Show `apt` failing, and read its proposed solutions. Then solve the same constraint set three ways: virtual packages/`Provides`, versioned coexistence (different sonames), and containers. Finally, install Nix and show the same conflict simply **not existing**.
    *Acceptance:* the reproducible conflict; `apt` and `aptitude` output (do they propose different solutions? report it); the three resolutions working; the Nix demonstration; and a `WHY.md` explaining why this is NP-complete and what `libsolv` actually does about it.

## 13. Key takeaways

- **Two layers, always.** Low-level (`dpkg`/`rpm`) installs one file and *checks* dependencies; high-level (`apt`/`dnf`) *resolves* them from repositories and calls the low-level tool. `apt update` refreshes the **index**; `apt upgrade` installs.
- **A package is an archive + metadata + maintainer scripts that run as root.** Installing is executing arbitrary code with full privilege — which is why the signature chain is everything. `dpkg-deb -e` / `rpm -q --scripts` is your ninety-second due diligence.
- **Trust = a GPG-signed index containing package hashes.** HTTPS is not the control; **signing is**. `signed-by=` scopes a key to one repo — the whole reason `apt-key add` was deprecated, because a global keyring let any key sign for any repository.
- **Never `curl | sudo bash`, never `gpgcheck=0`, never `[trusted=yes]`.**
- **The package database is a tool, not trivia**: `dpkg -S`/`rpm -qf` ("where did this file come from?"), `dpkg -V`/`rpm -V` ("what's been modified?"), `apt-cache policy` ("why is apt doing that?"), `rdepends` ("can I remove this?").
- **Backporting means the version number lies.** `curl 7.81.0-1ubuntu1.14` can contain an 8.4.0 fix. Use `apt changelog` and the **distro's security tracker**, not upstream version ranges — this is the source of most CVE-scanner false positives.
- **Dependency resolution is NP-complete** (it's SAT); modern resolvers use SAT solvers. Lockfiles, virtual packages, containers, and Nix are all responses to this.
- **Dockerfiles:** `update && install` in one `RUN`, `--no-install-recommends`, `rm -rf /var/lib/apt/lists/*`, `DEBIAN_FRONTEND=noninteractive`. Pin **and** automate the bumps.
- **Never `sudo pip install`** alongside distro packages; **`/usr/local`** for anything you build, never `/usr`. `apt-get` in scripts, `apt` interactively.
- **Conffile prompts hang automation** — `--force-confold` + `--force-confdef`. RPM's `.rpmnew` files are the quieter, arguably worse failure.
- **Alpine is not automatically smaller** for Python — musl means no `manylinux` wheels, so pip compiles from source.
