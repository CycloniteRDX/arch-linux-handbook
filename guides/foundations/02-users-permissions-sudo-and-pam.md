# Users, permissions, sudo, and PAM

## Purpose and scope

A Linux login joins several mechanisms that are easy to confuse:

- the account supplies an identity;
- file ownership and mode bits control ordinary filesystem access;
- `sudo` delegates selected commands to another identity;
- PAM lets a program apply an authentication and session policy;
- polkit authorizes actions exposed by cooperating services;
- logind, udev, ACLs, and device-specific policy can grant access without a
  permanent group membership.

This guide explains how these layers interact in this project. It does not
replace the dedicated future guides for polkit, portals, or complete PAM module
reference.

## Identity: names are labels for numbers

The kernel makes access decisions using numeric user and group identifiers:
UIDs and GIDs. Names such as `neon`, `root`, and `wheel` are the readable
mapping presented by user-space tools.

An ordinary login has:

- one UID;
- one primary GID;
- zero or more supplementary GIDs;
- a home directory and login shell recorded in the account database.

Inspect the current identity without modifying it:

```bash
id
id neon
getent passwd neon
getent group wheel
```

`/etc/passwd` contains account metadata, not the password hash. Local password
hashes and aging fields are stored in root-readable `/etc/shadow`. Do not copy
either file into a public repository; even non-secret account metadata can
reveal machine-specific information.

The runbook creates the project account with:

```bash
useradd -m -U -G wheel -s /bin/bash neon
```

Here `-m` creates the home directory, `-U` creates a primary group named
`neon`, `-G wheel` adds one supplementary group, and `-s` chooses Bash as the
login shell. The password is set interactively with `passwd`; it is never
placed on a command line or in Git.

When changing supplementary groups later, remember that a running process
keeps the group list it inherited at login. Log out and start a fresh login
session before judging the result.

## Ownership and mode bits answer different questions

Every ordinary filesystem object has an owning UID, an owning GID, and mode
bits. `chown` changes ownership; `chmod` changes the mode. One does not imply
the other.

Inspect both, including the numeric identifiers:

```bash
ls -ld /path/to/object
stat /path/to/object
stat -c '%A %a %U:%G %u:%g %n' /path/to/object
```

The familiar three permission classes are:

| Class | Symbol | Who it describes |
| --- | --- | --- |
| User | `u` | The file's owning UID. |
| Group | `g` | Processes whose groups include the file's owning GID. |
| Other | `o` | Everyone not matched by the first two classes. |

Within each class, `r`, `w`, and `x` mean read, write, and execute. Their octal
values are 4, 2, and 1, so `0644` means `rw-r--r--` and `0755` means
`rwxr-xr-x`.

Prefer symbolic modes when expressing a narrow change:

```bash
chmod u+x script.sh
chmod go-rwx private-directory
```

Prefer a complete numeric mode when the exact final policy matters:

```bash
chmod 0700 private-directory
chmod 0600 private-file
sudo chmod 0440 /etc/sudoers.d/10-wheel
```

The leading zero makes the octal intent explicit. It also leaves room for the
special setuid, setgid, and sticky bits, which are not needed for routine files
in this project.

## Directory permissions are not file permissions

For a regular file:

- `r` permits reading its contents;
- `w` permits changing its contents;
- `x` permits attempting to execute it.

For a directory:

- `r` permits listing its names;
- `w` permits creating, removing, or renaming entries, when traversal is also
  allowed;
- `x` permits traversing or searching the directory.

This explains two otherwise surprising cases. A process may know a filename
but be unable to reach it because a parent directory lacks `x`. Conversely,
deleting a file is principally controlled by the parent directory, so a user
may be able to remove a read-only file from a writable directory.

Access also depends on every parent directory, mount options, read-only
filesystems, ACLs, immutable attributes, and program-specific restrictions.
Mode bits alone are not always the whole answer.

Useful diagnostics are:

```bash
namei -l /complete/path/to/file
getfacl -p /complete/path/to/file
lsattr /complete/path/to/file
findmnt -T /complete/path/to/file
```

Use these only as needed; do not respond to a permission error by immediately
running `chmod -R 777` or `sudo chown -R`. Both commands erase useful policy
over an unbounded tree and can create security or package-integrity problems.

## Default permissions and umask

Programs request an initial mode when creating an object. The process umask
removes bits from that requested mode; it does not add permissions.

Inspect the current shell's mask in symbolic and octal form:

```bash
umask -S
umask
```

With a common umask of `0022`, a program requesting `0666` for a regular file
normally produces `0644`, while a requested `0777` directory normally becomes
`0755`. A program may deliberately request a stricter mode or change it later,
so this is not a guarantee that every new file has those values.

Set a different umask only with a defined scope and reason. A global change can
affect services, build tools, shared directories, and files created long after
the original task.

## Groups do not grant powers by themselves

A group is only a named GID. Membership matters when another policy refers to
that group: file mode bits, an ACL, a sudoers rule, or an application's own
configuration.

Membership in `wheel` therefore does not inherently grant administrative
access. The runbook creates an explicit sudoers rule that gives `wheel` its
project role. Similarly, this installation does not add the user to legacy
groups such as `audio`, `video`, `storage`, or `input` pre-emptively. Modern
desktop device access is commonly granted to the active local session through
udev, logind, ACLs, or a broker service. Add a persistent group only when a
specific documented use case requires it.

## What sudo actually delegates

`sudo` evaluates its policy and, when allowed, executes a command with another
identity. The default target is usually root. Authentication normally asks for
the invoking user's password; it is not a request for the root password.

The project stores this rule in `/etc/sudoers.d/10-wheel`:

```sudoers
%wheel ALL=(ALL:ALL) ALL
```

Read from left to right:

| Field | Meaning in this rule |
| --- | --- |
| `%wheel` | Match members of the `wheel` group. |
| First `ALL` | Match every host in the sudoers host field. |
| `(ALL:ALL)` | Permit any target user and target group. |
| Final `ALL` | Permit any command. |

This is broad administrative delegation, protected by password authentication.
It is appropriate for the sole owner-administrator profile fixed by the
runbook, but it should not be copied blindly to a multi-user or restricted
machine.

The filename `10-wheel` is a local organization choice, not a sudo priority
level. Sudoers is a policy language, not a generic last-file-wins configuration
system. Ordering, aliases, tags, and matching rules must be interpreted using
`sudoers(5)`.

## Why use visudo

Edit sudo policy with `visudo`, including files under `/etc/sudoers.d`:

```bash
sudo EDITOR=micro visudo -f /etc/sudoers.d/10-wheel
sudo chmod 0440 /etc/sudoers.d/10-wheel
sudo visudo -c
```

`visudo` locks the target against simultaneous edits and checks syntax before
installing the change. `visudo -c` validates the complete policy and reports
the included files it parses.

Keep an authenticated root shell open while introducing the first sudo rule.
Test from a separate login as the regular user:

```bash
sudo -k
sudo -v
sudo -l
sudo whoami
```

`sudo -k` invalidates the invoking user's cached timestamp, `sudo -v` validates
credentials without running the intended administrative command, and
`sudo -l` shows the resulting permissions. `sudo whoami` should print `root`
for this project policy.

Do not add `NOPASSWD` merely for convenience. A narrow automated task may
justify a carefully constrained rule later, but an unrestricted passwordless
rule removes an important boundary for any process already running as the
user.

## PAM is a stack used by applications

PAM, the Pluggable Authentication Modules framework, lets an application use a
named policy without implementing password verification and session setup by
itself. Files under `/etc/pam.d/` define service-specific stacks.

The four PAM management groups answer different questions:

| Type | Role |
| --- | --- |
| `auth` | Establish identity or obtain authentication credentials. |
| `account` | Decide whether the authenticated account may be used now. |
| `password` | Change or update an authentication token. |
| `session` | Perform work when opening or closing a session. |

Each line also has a control field. Common simple controls are:

- `required`: failure ultimately fails the stack, but later modules continue;
- `requisite`: failure normally returns immediately;
- `sufficient`: success can complete the group early when no earlier required
  module failed;
- `optional`: its result usually does not decide the stack when other modules
  provide a decisive result.

“Optional” does not mean “never runs” or “can never matter”. Modules execute in
order, can consume authentication tokens, start helpers, and have side effects.
The complete stack and the PAM application's behavior determine the result.
Advanced bracketed controls, `include`, and `substack` should be read as PAM
policy, not guessed from their names.

## PAM in this workstation

The project touches PAM in three distinct situations:

1. Console `login` authenticates the user and opens the original session.
2. greetd presents a graphical login path and invokes its own PAM service;
   tuigreet is the interface, not the password verifier.
3. swaylock asks PAM to authenticate the already logged-in user; it does not
   create a new login session or grant sudo privileges.

GNOME Keyring adds optional hooks to `/etc/pam.d/login` and
`/etc/pam.d/passwd`. The `auth` and `session` hooks receive the login token and
start or unlock the login keyring, while the `password` hook keeps its password
synchronized when the account password changes. These hooks do not replace the
module that authenticates the console login.

Inspect which package supplied a PAM service and review the active text:

```bash
pacman -Qo /etc/pam.d/swaylock
sudo sed -n '1,200p' /etc/pam.d/login
sudo sed -n '1,200p' /etc/pam.d/swaylock
```

Do not publish the output of authentication logs or account databases without
reviewing it for usernames, paths, and other machine information.

## PAM changes require a live recovery path

A syntactically plausible PAM change can still prevent login. Before editing:

1. keep the current root or working administrative session open;
2. make a metadata-preserving backup of each exact file;
3. change only the intended lines;
4. test a second TTY login before closing the first session;
5. test swaylock and greetd separately, because they use different PAM service
   names.

The post-install procedure follows this rule for GNOME Keyring. If console
login fails, restore the reviewed backups from the still-open session or the
Arch ISO/chroot recovery path, then test again. Do not reboot merely to test a
PAM edit when a second TTY can prove whether login still works.

## Sudo, PAM, and polkit are not interchangeable

| Mechanism | Main question | Typical project example |
| --- | --- | --- |
| File modes and ACLs | May this process access this filesystem object? | Root-only Secure Boot material. |
| `sudo` | May this user run this command as another identity? | `sudo pacman -Syu`. |
| PAM | How does this application authenticate and manage an account/session? | login, greetd, swaylock, GNOME Keyring hooks. |
| polkit | May this subject request this privileged service action? | A graphical application asking a system service to perform an operation. |
| XDG Desktop Portal | How does an application request a desktop-mediated capability? | File chooser, screenshot, or screen-cast interface. |

A graphical polkit authentication agent may display a password dialog, but it
does not become `sudo` and is not itself the policy engine. Likewise, a portal
prompt does not automatically mean root privilege. The dedicated workstation
guide will trace these components and their processes in detail.

## A safe diagnostic sequence

When an operation reports “permission denied”, identify the boundary before
changing it:

```bash
id
namei -l /complete/path
stat /complete/path
getfacl -p /complete/path
findmnt -T /complete/path
```

Then ask:

1. Is this a filesystem access failure, an application policy denial, a PAM
   authentication failure, a sudoers denial, or a polkit denial?
2. Which identity and groups does the failing process actually have?
3. Is the target file package-owned, locally administered, or user-owned?
4. Does the journal name the denying component?
5. What is the smallest reversible change that expresses the intended access?

Use `journalctl` for the specific unit or boot and avoid pasting an entire
unfiltered journal into a public issue.

## Recovery principles

- A broken sudoers rule is repaired from the retained root session or an Arch
  ISO/chroot, then checked with `visudo -c`.
- A broken PAM stack is restored from its exact backup before experimenting
  further.
- Incorrect ownership is repaired only after identifying the intended UID/GID;
  do not recursively chown a system tree.
- Incorrect modes should be compared with a known-good file, package metadata,
  or the program manual. `pacman -Qkk package` can help identify altered
  package files but does not decide every local policy.
- A group change is verified in a new login session rather than patched around
  with wider file permissions.

## Sources

- [chmod(1)](https://man.archlinux.org/man/chmod.1)
- [chown(1)](https://man.archlinux.org/man/chown.1)
- [useradd(8)](https://man.archlinux.org/man/useradd.8)
- [usermod(8)](https://man.archlinux.org/man/usermod.8)
- [sudoers(5)](https://man.archlinux.org/man/sudoers.5)
- [visudo(8)](https://man.archlinux.org/man/visudo.8)
- [pam(8)](https://man.archlinux.org/man/pam.8)
- [pam.d(5)](https://man.archlinux.org/man/pam.d.5)
- [ArchWiki: Users and groups](https://wiki.archlinux.org/title/Users_and_groups)
- [ArchWiki: File permissions and attributes](https://wiki.archlinux.org/title/File_permissions_and_attributes)
- [ArchWiki: Sudo](https://wiki.archlinux.org/title/Sudo)
- [ArchWiki: PAM](https://wiki.archlinux.org/title/PAM)
