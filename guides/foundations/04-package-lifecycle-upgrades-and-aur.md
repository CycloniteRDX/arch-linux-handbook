# Package lifecycle, upgrades, and the AUR

## Purpose and scope

Arch Linux is maintained as one rolling system. Pacman does not merely download
files: it resolves package relationships, verifies packages, executes one
transaction, updates its local database, preserves selected configuration, and
runs package hooks.

This guide explains how to inspect and maintain that state without creating a
partial upgrade or treating package cleanup as an end in itself. It covers:

- official repositories and local package state;
- complete upgrades and Arch News;
- package queries, ownership, install reasons, dependencies, and orphans;
- configuration files, cache, logs, mirrors, and interrupted transactions;
- the trust and maintenance boundary of the Arch User Repository.

Building original packages for publication deserves a separate packaging
guide. Here the AUR is considered only from the workstation user's side.

## Pacman coordinates several kinds of state

The important locations have different purposes:

| Location | Role |
| --- | --- |
| `/etc/pacman.conf` | Repository, signature, cache, hook, and transaction policy. |
| `/etc/pacman.d/mirrorlist` | Ordered server URLs used by official repositories. |
| `/var/lib/pacman/local/` | Installed-package database and recorded metadata. |
| `/var/lib/pacman/sync/` | Local copies of enabled repository databases. |
| `/var/cache/pacman/pkg/` | Downloaded package archives retained for reuse or recovery. |
| `/var/log/pacman.log` | Pacman transaction history and messages. |
| `/etc/pacman.d/gnupg/` | Pacman's package-signing keyring and trust database. |
| `/etc/pacman.d/hooks/` | Local transaction hooks, when deliberately added. |

The files installed on `/usr`, `/etc`, and elsewhere are only one part of the
state. Manually copying or deleting a package-owned file does not update
pacman's database. Conversely, editing the database does not repair files on
disk. Avoid `--dbonly`, `--nodeps`, `-Rdd`, and `--overwrite` unless a specific
recovery procedure explains exactly why the normal consistency checks must be
bypassed.

Inspect pacman's active paths and configuration source with:

```bash
pacman -v
```

Do not expose the output publicly without reviewing custom repository URLs,
paths, or other local information.

## Packages, repositories, groups, and virtual provisions

An official repository database describes package names, versions,
architectures, dependencies, conflicts, replacements, groups, checksums, and
signatures. A package archive contains the files and metadata to install.

Three concepts are easily confused:

- a package is an installable archive tracked as one package database entry;
- a package group is a named selection of otherwise independent packages;
- a meta-package is an ordinary package whose dependencies pull in a selected
  set, often with few or no payload files of its own.

A dependency may also name a virtual capability supplied by more than one
package. Pacman can ask which provider to choose. Read provider, replacement,
and conflict prompts instead of accepting them automatically.

Inspect repository and installed views separately:

```bash
pacman -Si package-name
pacman -Qi package-name
pacman -Sg group-name
```

`-Si` reads synchronized repository metadata; `-Qi` reads local installed
metadata. Their versions can differ when an update is pending.

## One rolling system means complete upgrades

The canonical operation is:

```bash
sudo pacman -Syu
```

The flags express one coherent transaction:

- `-S` selects synchronization with repositories;
- `-y` refreshes enabled repository databases;
- `-u` upgrades every installed package for which the synchronized repositories
  provide a newer applicable version.

Installing a new official package can be included in the same transaction:

```bash
sudo pacman -Syu package-name
```

Do not use `pacman -Sy` as a standalone maintenance action or follow it with a
package installation. It refreshes the repository view without upgrading
installed libraries and applications. A later `pacman -S package-name` may then
install a new package built against versions the machine does not have,
producing an unsupported partial upgrade.

Running `pacman -Sy package-name` is not safer: it combines the same database
refresh with installation while omitting the full `-u` upgrade.

If an upgrade refreshes databases and then fails, diagnose and complete the
full upgrade before installing or removing unrelated packages. Repeatedly
forcing individual packages through usually increases the inconsistency.

## Read Arch News before changing the system

Package metadata and hooks cannot safely automate every transition. The
[official Arch News](https://archlinux.org/news/) publishes manual intervention
when an upgrade needs administrator action.

Before upgrading:

1. identify news posts newer than the last successful upgrade;
2. read the complete instructions relevant to installed packages;
3. ensure recovery media and enough troubleshooting time are available;
4. avoid a large upgrade immediately before class, travel, or a deadline.

The project does not install an unreviewed AUR hook to automate this trust
decision. RSS or another notification mechanism may remind the user that news
exists, but cannot decide whether the instructions apply.

## Check available updates without altering the live sync database

`checkupdates`, supplied by `pacman-contrib`, uses a separate temporary database
to compare the installed system with current repository state:

```bash
checkupdates
```

It prints installed and available versions. No output with exit status 2 means
that no official updates are available; it is not a transaction failure.

This is safer for an informational check than `pacman -Sy` followed by
`pacman -Qu`, because it does not leave the system's normal sync database ahead
of the installed package set. It still does not install anything, read Arch
News, merge configuration, or verify that a later reboot succeeds.

## Read the transaction before confirming it

For each `pacman -Syu`, inspect:

- packages to upgrade, install, replace, or remove;
- download and installed-size changes;
- dependency, provider, conflict, and replacement prompts;
- signature or keyring errors;
- post-transaction hooks and manual messages;
- creation of `.pacnew`, `.pacsave`, or `.pacorig` files;
- UKI, initramfs, systemd-boot, and Secure Boot signing results when relevant.

Do not make `--noconfirm` the normal workstation workflow. It suppresses the
point at which an unexpected removal, provider change, or manual intervention
can be stopped and investigated.

Package signatures authenticate packages according to pacman's configured
trust policy. HTTPS mirrors protect the transport and improve privacy, but
HTTPS is not a substitute for package signatures. Conversely, a valid package
signature says who signed the package; it does not make third-party source or
an AUR build recipe inherently trustworthy.

## Query before changing

Pacman's operations describe which database is being queried:

| Command | Question |
| --- | --- |
| `pacman -Q package` | Is this package installed, and at which version? |
| `pacman -Qi package` | What installed metadata and dependencies are recorded? |
| `pacman -Qii package` | Also show backup-file state recorded for the package. |
| `pacman -Ql package` | Which paths belong to this installed package? |
| `pacman -Qo /path` | Which installed package owns this existing path? |
| `pacman -Qk package` | Are all recorded package files present? |
| `pacman -Qkk package` | Also compare detailed metadata when available. |
| `pacman -Qs terms` | Search installed package names and descriptions. |
| `pacman -Ss terms` | Search synchronized repository metadata. |
| `pacman -Qe` | List packages marked explicitly installed. |
| `pacman -Qd` | List packages marked installed as dependencies. |
| `pacman -Qdt` | List dependency packages no longer required. |
| `pacman -Qm` | List foreign packages absent from enabled sync databases. |

`pacman -Qo` reports database ownership, not who last modified a file.
`pacman -Qkk` can expose missing or changed package metadata, but changes to
files deliberately marked as backup configuration are not automatically
mistakes.

To discover which official repository package contains a file that is not
installed, refresh the separate file database and query it:

```bash
sudo pacman -Fy
pacman -F bin/example-command
```

`-F` searches repository file databases; `-Qo` searches installed ownership.

## Explicit packages, dependencies, and optional dependencies

Pacman records an install reason:

- explicit means the package was selected as a wanted top-level package;
- dependency means it was installed to satisfy another package.

The reason affects later orphan detection and recursive removal, not whether
the package can run. Inspect it with `pacman -Qi` or the `-Qe` and `-Qd`
filters.

If the recorded reason is genuinely wrong, correct only the metadata:

```bash
sudo pacman -D --asexplicit package-name
sudo pacman -D --asdeps package-name
```

Do not change reasons merely to make an orphan list empty. First identify the
intended top-level applications and why each candidate remains installed.

Optional dependencies add features but are not required for the base package
dependency graph. `pacman -Qdt` conservatively excludes packages still needed
as optional dependencies; using the unrequired filter twice broadens the list
to include packages required only optionally. Broader output is not an
automatic removal list.

## Orphans are candidates, not rubbish

Inspect orphaned dependency packages:

```bash
pacman -Qdt
```

An orphan can result from removing an application, changing providers, or
building a package. It may also be a tool the user now wants independently but
never marked explicit.

For each candidate:

```bash
pacman -Qi package-name
pactree -r package-name
```

The reverse tree is only exploratory and may be noisy; `pactree` comes from
`pacman-contrib`. Prefer inspecting one package at a time over constructing a
blind removal pipeline.

When removal is truly intended, preview pacman's complete transaction. `-Rns`
combines removal, recursive removal of now-unneeded dependency packages, and
suppression of `.pacsave` creation. That last property makes it unsuitable as a
default reflex when configuration may be worth preserving.

Never pipe `pacman -Qdtq` directly into a privileged removal command without
review. An empty list, shell word handling, or a newly orphaned but useful tool
can make automation surprising.

## Configuration files and pacnew

Packages can designate selected files for backup handling. On upgrade, pacman
compares the old packaged version, the installed file, and the new packaged
version.

Important outcomes include:

- an unmodified installed file can be replaced by the new packaged file;
- a locally modified file can remain active;
- when both local and packaged versions changed incompatibly, the new packaged
  file is written as `.pacnew` for manual merging;
- removal can preserve modified backup configuration as `.pacsave`, unless the
  chosen removal options suppress it;
- `.pacorig` can preserve a pre-existing file encountered during installation.

List unresolved auxiliary files with:

```bash
sudo pacdiff --output
```

For each result:

1. identify the owning package and affected program;
2. compare the active and new versions;
3. preserve intentional local policy;
4. incorporate new required directives or syntax;
5. validate the merged configuration;
6. reload or restart only what the program requires;
7. remove the auxiliary file only after the merge is complete.

Do not accept every `.pacnew` wholesale and do not leave them indefinitely.
A drop-in can reduce direct conflicts with a main file, but a package update
may still rename an option or change the default that the drop-in overrides.
See the [configuration precedence guide](01-configuration-files-and-drop-ins.md)
for the wider model.

## Package cache and local recovery

Downloaded archives remain under `/var/cache/pacman/pkg` unless cleaned. This
uses disk space but offers useful properties:

- reinstall a cached package without downloading it again;
- compare or inspect previous package archives;
- perform a targeted emergency downgrade when the old archive is still
  compatible with the rest of the system.

Preview conservative cleanup:

```bash
sudo paccache --dryrun
```

The project enables the packaged weekly timer:

```bash
sudo systemctl enable --now paccache.timer
```

Its default service retains the three most recent cached versions of each
package. This balances recovery value and disk growth without adding a custom
hook after every transaction.

Avoid `pacman -Scc` for routine maintenance: it removes the convenient local
archive history. More aggressive cleanup is justified only by real storage
pressure and an understood recovery alternative.

A cached archive can be installed with `pacman -U`, but downgrading one package
does not roll back its dependencies, configuration migrations, database
formats, kernel modules, or hooks. Treat a downgrade as a temporary diagnosed
repair, not a general snapshot mechanism. Complete backups and restore drills
remain necessary.

## Mirrors provide transport, not package policy

`/etc/pacman.d/mirrorlist` supplies ordered servers for official repository
content. Pacman tries configured servers according to its mirror policy; a
closer server is not necessarily faster or more synchronized.

Inspect active entries:

```bash
grep '^Server' /etc/pacman.d/mirrorlist
```

If downloads are reliable and reasonably fast, an existing list does not need
constant rewriting. A long list is not itself a problem.

When mirrors are demonstrably stale, unavailable, or slow, the project allows
a deliberate Reflector run after backing up the current list:

```bash
sudo cp -a /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.before-reflector
sudo reflector --country Spain,Portugal,France,Germany --age 24 --protocol https --latest 20 --sort rate --save /etc/pacman.d/mirrorlist
grep '^Server' /etc/pacman.d/mirrorlist
```

The result must contain multiple usable servers and should be inspected before
the next complete upgrade. If it is unsuitable, restore the exact backup.

This profile does not enable `reflector.timer`. Automatic mirror rewriting can
change a working input immediately before maintenance and provides little
value for these laptops compared with a reviewed, on-demand update.

## The AUR is a recipe collection, not an official binary repository

The Arch User Repository hosts user-contributed package build metadata. An AUR
entry normally provides a `PKGBUILD`, `.SRCINFO`, and possibly patches or an
`.install` script. It is not an official Arch package and is not reviewed or
supported like packages in the official repositories.

For a required AUR-only component:

1. install the official `base-devel` tools through a complete upgrade;
2. obtain the package build files as the regular user;
3. inspect `PKGBUILD`, `.SRCINFO`, `.install`, patches, source URLs, checksums,
   and signature handling;
4. confirm that the upstream project and fetched sources are the intended ones;
5. build with `makepkg` as the regular user, never as root;
6. install the resulting package archive through pacman;
7. retain enough information to reproduce, update, or remove it;
8. track all foreign packages with `pacman -Qm`.

A `PKGBUILD` is executable shell code. Reading only `.SRCINFO`, votes, comments,
or the package name is not a security review. Build functions, source arrays,
prepare steps, install scripts, patches, and dynamically obtained content can
all change what runs or enters the final package.

Build in a clean, disposable environment when the package or dependency set is
important enough to justify the extra isolation. Building directly on the
workstation can leave make dependencies installed, consume untracked space,
and inherit local environment differences.

## AUR helpers do not move the trust boundary

An AUR helper can automate searching, fetching, dependency resolution,
rebuilding, and installation. It cannot decide whether a new build recipe or
changed source is trustworthy.

The project therefore does not install a helper pre-emptively. Learn one manual
build and review cycle first. If a helper is later selected:

- understand when it refreshes official databases and ensure upgrades remain
  complete;
- make it show PKGBUILD changes before building;
- do not allow unattended confirmation of arbitrary recipes;
- distinguish official upgrades from AUR rebuilds in logs and diagnosis;
- retain a recovery path that does not depend on the helper itself.

Popularity and votes indicate usage, not safety. A valid upstream signature
can authenticate a source release but does not validate every command in the
PKGBUILD around it.

## Foreign packages require continuing maintenance

List packages absent from enabled official sync databases:

```bash
pacman -Qm
```

The list may include AUR builds, manually installed local packages, and
packages removed from or moved out of the enabled repositories. “Foreign” does
not prove that a package came from the AUR or that it is malicious.

Pacman can track a foreign package's installed files and dependency metadata,
but `pacman -Syu` cannot fetch a new build from the AUR. The user remains
responsible for monitoring recipe changes, upstream security releases,
rebuilding when required, and removing abandoned software.

Foreign binary packages may need rebuilding after official shared-library or
toolchain transitions. Diagnose missing libraries with package metadata and
the actual loader error; do not solve it by creating arbitrary compatibility
symlinks between library versions.

## Logs, hooks, and boot artifacts

Review recent package history with:

```bash
sudo tail -n 200 /var/log/pacman.log
```

The log helps establish which transaction installed, upgraded, or removed a
package. It does not capture every later manual file edit or every application
failure.

Transaction hooks may rebuild initramfs or UKIs, update caches, reload systemd,
or perform package-specific actions. A transaction is not complete merely
because archives were unpacked; read every hook result.

When kernel, microcode, mkinitcpio, systemd, systemd-boot, or sbctl changes,
verify the boot entries and signatures before rebooting:

```bash
sudo bootctl list
sudo sbctl verify
```

Do not disable Secure Boot to hide a failed signing or generation hook. Repair
the artifact while the currently running system still provides a recovery
shell.

## Interrupted transactions and the database lock

Pacman uses `/var/lib/pacman/db.lck` to prevent concurrent package database
writers. A lock error usually means another pacman, helper, or package frontend
is running.

Do not immediately delete the lock. First identify active processes and allow a
real transaction or hook to finish:

```bash
ps -ef | grep -E '[p]acman|[m]akepkg'
sudo lsof /var/lib/pacman/db.lck 2>/dev/null
```

Only when no package operation remains and the prior process is known to have
terminated abnormally should a stale lock be considered for removal. Then
inspect `/var/log/pacman.log`, rerun the complete upgrade, and resolve its exact
errors. A missing lock file does not prove that package files and database state
are consistent.

Do not run two package managers concurrently. A graphical frontend, AUR helper,
or background script still shares pacman's database and transaction boundary.

## Recovery after a problematic upgrade

Use evidence before downgrading:

1. read Arch News and the complete pacman transaction output;
2. inspect `/var/log/pacman.log` and relevant service or boot journals;
3. resolve `.pacnew` and changed syntax;
4. verify whether the failing file belongs to a package with `pacman -Qo`;
5. check package files with `pacman -Qk` or `-Qkk` where appropriate;
6. verify UKIs and signatures before a reboot;
7. use the Arch ISO/chroot path when the installed system cannot boot.

Reinstalling the current official package can restore missing package-owned
files, but it does not erase every local configuration file and should not be
used to overwrite unexplained conflicts blindly.

Avoid long-term `IgnorePkg` or partial downgrades as a routine way to freeze a
rolling system. Temporarily holding a package may be part of a documented
transition, but it creates a divergence that must be tracked and resolved.

## Project maintenance sequence

The canonical workstation cycle is:

1. Ensure recovery media, LUKS access, backups, and troubleshooting time are
   available.
2. Read new Arch News entries.
3. Optionally inspect official updates with `checkupdates`.
4. Run `sudo pacman -Syu` and read the transaction and hook output.
5. Resolve every result from `sudo pacdiff --output`.
6. Inspect failed systemd units and new journal errors.
7. Verify UKIs and signatures when boot-related packages changed.
8. Review and rebuild foreign packages separately when needed.
9. Restart affected services or reboot at a controlled time.
10. Confirm the current system, desktop, network, storage, and backup behavior.

The project automates conservative cache cleanup, not package upgrade approval.

## Sources

- [pacman(8)](https://man.archlinux.org/man/pacman.8)
- [pacman.conf(5)](https://man.archlinux.org/man/pacman.conf.5)
- [checkupdates(8)](https://man.archlinux.org/man/checkupdates.8)
- [paccache(8)](https://man.archlinux.org/man/paccache.8)
- [pacdiff(8)](https://man.archlinux.org/man/pacdiff.8)
- [makepkg(8)](https://man.archlinux.org/man/makepkg.8)
- [PKGBUILD(5)](https://man.archlinux.org/man/PKGBUILD.5)
- [reflector(1)](https://man.archlinux.org/man/reflector.1)
- [Arch Linux News](https://archlinux.org/news/)
- [ArchWiki: System maintenance](https://wiki.archlinux.org/title/System_maintenance)
- [ArchWiki: Pacman](https://wiki.archlinux.org/title/Pacman)
- [ArchWiki: Mirrors](https://wiki.archlinux.org/title/Mirrors)
- [ArchWiki: Reflector](https://wiki.archlinux.org/title/Reflector)
- [ArchWiki: Arch User Repository](https://wiki.archlinux.org/title/Arch_User_Repository)
- [ArchWiki: Makepkg](https://wiki.archlinux.org/title/Makepkg)
