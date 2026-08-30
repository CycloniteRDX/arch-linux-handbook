# Systemd units, activation, and the journal

## Purpose and scope

Systemd is more than a command used to start daemons. The system instance runs
as PID 1, brings up userspace, models resources as units, orders work, tracks
processes in control groups, and records structured events in the journal.
Separate systemd instances manage services for logged-in users.

This guide builds a practical model for answering:

- what an installed unit represents;
- why it is running;
- whether it will be pulled in again;
- what starts it on demand;
- where its effective definition comes from;
- what happened when it failed.

It uses the units selected by this Arch workstation, but the concepts apply to
other systemd systems.

## Units are the objects systemd manages

A unit name combines an identifier and a type suffix. The suffix tells systemd
what kind of object it represents.

| Type | Represents | Project example |
| --- | --- | --- |
| `.service` | A process or set of supervised processes. | `NetworkManager.service`, `firewalld.service`, `greetd.service` |
| `.socket` | A listening IPC or network socket that can activate a service. | `cups.socket`, `systemd-rfkill.socket` |
| `.timer` | A monotonic or calendar trigger for another unit. | `fstrim.timer`, `paccache.timer`, `fwupd-refresh.timer` |
| `.target` | A grouping or synchronization point. | `multi-user.target`, `graphical.target` |
| `.mount` | A filesystem mount. | Units generated from `/etc/fstab` |
| `.automount` | An on-demand mount point. | Optional automount designs |
| `.swap` | A swap device or file. | Disk swap generated from configuration |
| `.device` | A kernel device known to systemd. | Storage-device dependencies |
| `.path` | A filesystem path whose state can trigger a unit. | Not canonical in this project |
| `.slice` | A resource-control grouping in the cgroup tree. | `system.slice`, `user.slice` |
| `.scope` | Processes created elsewhere but tracked by systemd. | Login sessions and transient scopes |

Not every unit has a persistent file with the same name. Units may be generated
at boot, synthesized from kernel state, or created transiently over D-Bus.

Inspect a known unit and its source:

```bash
systemctl status firewalld.service --no-pager
systemctl show -p FragmentPath -p SourcePath -p LoadState firewalld.service
systemctl cat firewalld.service
```

`systemctl cat` shows the main unit file and applied drop-ins. It does not prove
that the service's own application configuration is valid.

## Loaded, active, and enabled are different dimensions

Systemd reports several independent kinds of state.

### Load state

The load state says whether systemd found and parsed a definition. Common
values include `loaded`, `not-found`, `bad-setting`, `error`, and `masked`.

### Active state

The active state describes runtime now:

- `active`: the unit currently fulfils its type-specific active condition;
- `inactive`: it is not active;
- `activating` or `deactivating`: a transition is in progress;
- `failed`: a job or supervised process failed;
- `reloading`: an active unit is reloading its application configuration.

“Active” does not always mean “a daemon process is running”. A timer is active
while waiting, a mount is active while mounted, and a successful oneshot service
may be considered active after its process exited when `RemainAfterExit=yes`.

### Unit-file state

The unit-file state describes installation and enablement relationships. Common
results from `systemctl is-enabled` include:

- `enabled`: persistent links pull the unit into another unit;
- `enabled-runtime`: equivalent links exist only under `/run`;
- `disabled`: no enablement links currently pull it in;
- `static`: the unit has no install instructions and is normally started by a
  dependency or explicit request;
- `indirect`, `alias`, or `generated`: other forms that are not equivalent to
  ordinary enabled/disabled policy;
- `masked`: the unit name is linked to `/dev/null`, blocking activation.

Therefore these combinations are all possible:

| Enabled? | Active? | Possible explanation |
| --- | --- | --- |
| Yes | Yes | It was pulled in at boot and is running. |
| Yes | No | It has not reached its trigger, was stopped, finished, or failed. |
| No | Yes | It was started manually or activated by another mechanism. |
| No | No | It is installed but neither running nor enabled. |

Audit both dimensions explicitly:

```bash
systemctl is-enabled firewalld.service
systemctl is-active firewalld.service
systemctl status firewalld.service --no-pager
```

Do not use a single status word as a complete security or boot audit.

## Enable is not start

`enable` creates the links described by a unit's `[Install]` section so another
unit can pull it into a future transaction. It does not necessarily start the
unit now.

`start` creates a runtime job now. It does not necessarily arrange another
start after the next boot.

The combined form is deliberate:

```bash
sudo systemctl enable --now firewalld.service
```

Similarly:

```bash
sudo systemctl disable --now bluetooth.service
```

removes persistent enablement links and requests a stop. It does not guarantee
that nothing can activate the service later. A socket, timer, path, D-Bus
request, dependency, or explicit start may still pull in an unmasked unit.

Use `mask` only when activation must be refused:

```bash
sudo systemctl mask --now passim.service
systemctl is-enabled passim.service
```

Masking is stronger than disabling because it makes the unit definition
unloadable through its name. It can also break software that legitimately
depends on the unit. The project masks Passim for a documented network-surface
decision; masking is not a general cleanup technique.

Reverse a reviewed mask with:

```bash
sudo systemctl unmask passim.service
```

Unmasking does not enable or start it automatically.

## Dependencies and ordering are orthogonal

Systemd builds a transaction of jobs and their relationships before acting.
Two major relationship families answer different questions:

- requirement relationships ask which other units should be included;
- ordering relationships ask which job must reach its ordering point first.

For example:

- `Wants=bar.service` pulls in `bar` weakly;
- `Requires=bar.service` pulls it in more strongly;
- `After=bar.service` orders this unit after `bar` if both have jobs;
- `Before=bar.service` orders it before `bar` if both have jobs;
- `Conflicts=bar.service` expresses that both should not be active together.

`After=` alone does not start the named unit. `Requires=` alone does not imply
that startup must be sequential; without an ordering relationship the jobs may
run in parallel. Even `Requires=` does not mean that every later failure of the
required unit automatically stops the requiring unit; exact propagation also
depends on other directives and the job situation.

Inspect declared and reverse dependencies without editing them:

```bash
systemctl list-dependencies firewalld.service
systemctl list-dependencies --reverse firewalld.service
systemctl show -p Wants -p Requires -p After -p Before firewalld.service
```

Large dependency lists contain implicit and generated relationships. Treat
them as evidence to interpret, not a list of manual changes to reproduce.

## Targets group a boot state

A target normally groups units and provides an ordering milestone; it is not a
traditional daemon.

Important system targets include:

| Target | Purpose |
| --- | --- |
| `default.target` | Alias or selected entry point for the normal boot. |
| `multi-user.target` | Multi-user, non-graphical system baseline. |
| `graphical.target` | Graphical boot target layered on the multi-user baseline. |
| `rescue.target` | Restricted rescue environment with more system setup. |
| `emergency.target` | Minimal emergency shell and very little automatic setup. |

Inspect the configured default without changing it:

```bash
systemctl get-default
systemctl status default.target --no-pager
```

This project can use `graphical.target` while keeping TTY recovery available.
greetd is the graphical-login service selected by the post-install procedure;
the target itself is not the greeter.

`systemctl isolate target` stops units not required by the selected target and
can terminate the graphical session or network. Use rescue and emergency
isolation only with a recovery plan, preferably from a local console.

## Service units supervise processes

A service unit describes how a process is started, when systemd considers it
ready, how it is stopped, its environment and credentials, restart policy,
resource controls, and sandboxing.

Common service types include:

| `Type=` | Readiness model |
| --- | --- |
| `simple` | The `ExecStart` process is started; readiness is not explicitly signalled. |
| `exec` | Startup succeeds only after the service executable was invoked successfully. |
| `notify` | The service sends an explicit readiness notification. |
| `oneshot` | One or more startup commands run to completion before the unit is considered started. |
| `forking` | A traditional daemon forks and the parent exits; often paired with PID tracking. |

Do not assume `ExecStart=` is a shell command line. Systemd applies its own
tokenization and variable rules and does not automatically interpret pipes,
redirection, `&&`, or shell built-ins. When a shell is genuinely required, it
must be invoked explicitly, though a direct executable is usually easier to
audit.

Inspect the execution model:

```bash
systemctl show -p Type -p ExecStart -p MainPID -p SubState NetworkManager.service
systemctl status NetworkManager.service --no-pager
systemd-cgls --unit NetworkManager.service
```

Systemd tracks the unit's control group, not merely one PID file. Child
processes can therefore remain associated with the service even when the
application creates several processes.

## Reload, restart, and daemon-reload are different

| Command | What it asks to change |
| --- | --- |
| `systemctl reload name.service` | Ask the running application to reload its own configuration, if supported. |
| `systemctl restart name.service` | Stop and start the unit; runtime state and connections may be interrupted. |
| `systemctl try-restart name.service` | Restart only if it is already active. |
| `systemctl reload-or-restart name.service` | Reload when supported, otherwise restart. |
| `systemctl daemon-reload` | Make the systemd manager rerun generators and reload unit definitions. |
| `systemctl daemon-reexec` | Re-execute the manager itself while preserving state; rarely needed manually. |

Editing `/etc/firewalld/firewalld.conf` does not by itself require
`daemon-reload`, because that is application configuration rather than a unit
definition. Editing a unit file or its `.service.d` drop-in does require
`daemon-reload`, followed by whatever restart or reload the affected service
needs.

Conversely, `daemon-reload` does not cause a running process to reread its own
configuration and does not restart it.

## Socket activation

A `.socket` unit lets systemd own a listening socket before the corresponding
service is running. Incoming activity can activate the service and pass it the
already-open socket.

The post-install procedure enables `cups.socket` only when printing is needed:

```bash
sudo systemctl enable --now cups.socket
systemctl status cups.socket cups.service --no-pager
```

The socket may be active while `cups.service` is initially inactive. A print
request can then start the service. Disabling or stopping only the service does
not remove a still-active socket trigger.

Audit both systemd and the kernel's listening sockets:

```bash
systemctl list-sockets --all --no-pager
ss -lntup
```

A socket unit and a firewall solve different problems. The socket controls
whether something listens and can activate a service; the firewall controls
which network traffic may reach network sockets. A local Unix socket may not
appear in `ss -lntup`, which focuses on listening TCP and UDP endpoints.

## Timer activation

A timer activates another unit, normally a service with the same basename.
`fstrim.timer` therefore triggers `fstrim.service`; the timer does not perform
TRIM itself.

Timers may use:

- monotonic expressions such as time since boot or last activation;
- calendar expressions such as weekly schedules;
- `Persistent=yes` to catch up a missed calendar activation after the machine
  was powered off;
- accuracy and randomized-delay settings to coalesce wakeups or avoid many
  machines running work simultaneously.

Inspect the schedule and both paired units:

```bash
systemctl list-timers --all fstrim.timer --no-pager
systemctl cat fstrim.timer fstrim.service
systemctl status fstrim.timer fstrim.service --no-pager
journalctl -u fstrim.service --no-pager
```

An active timer is normally waiting. Its service may be inactive because it has
not run yet or because its last oneshot execution completed successfully.

This project uses timers deliberately:

- `paccache.timer` performs conservative weekly cache pruning;
- `fstrim.timer` performs periodic TRIM;
- `fwupd-refresh.timer` refreshes firmware metadata but does not install
  firmware automatically;
- `reflector.timer` remains disabled because mirror changes are manual.

Enabling a timer is not equivalent to approving every possible action related
to the application it supports.

## Generated units

Generators are short programs run early in manager startup and again during
`daemon-reload`. They translate other configuration sources into transient unit
files under generator directories in `/run`.

Examples relevant to this machine include:

- `/etc/fstab` translated into mount and swap units;
- `/etc/crypttab` or kernel command-line declarations translated into
  cryptsetup units where applicable;
- zram-generator configuration translated into
  `systemd-zram-setup@zram0.service` and related units.

This explains why editing a source file and reloading the manager can change
units that do not live permanently under `/etc/systemd/system`.

Inspect origin information instead of editing generated output:

```bash
systemctl show -p FragmentPath -p SourcePath systemd-zram-setup@zram0.service
systemctl cat systemd-zram-setup@zram0.service
```

Files under `/run/systemd/generator*` are transient products. Change the source
configuration and validate it; do not maintain local policy by editing generated
files.

## System manager and user manager

The system manager runs as PID 1 and controls machine-wide units. A user
manager runs for a logged-in user and controls that user's units and processes.

Compare the two scopes:

```bash
systemctl status firewalld.service --no-pager
systemctl --user status xdg-desktop-portal.service --no-pager
journalctl -b -u firewalld.service --no-pager
journalctl --user -b -u xdg-desktop-portal.service --no-pager
```

Do not use `sudo systemctl --user`: it normally targets root's user manager or
lacks the intended user's session environment. Run `systemctl --user` as the
logged-in user.

PipeWire, WirePlumber, portals, and related desktop services are user-session
components and may be socket- or D-Bus-activated. The project therefore does
not enable them as system services. Niri session startup and XDG autostart are
additional session mechanisms; not every desktop process must be converted
into a manually enabled user unit.

User lingering allows a user's manager to remain without an interactive login.
It is not enabled merely to make a desktop service start earlier, because that
would change session and power-lifecycle assumptions.

## The journal is structured evidence

The journal combines messages from units, processes, the kernel, boot records,
and other metadata. Start with a narrow question rather than grepping the
entire history.

Useful queries include:

```bash
journalctl -b -u firewalld.service --no-pager
journalctl -b -p warning..alert --no-pager
journalctl -b -1 -u systemd-suspend.service --no-pager
journalctl -k -b --no-pager
journalctl --since '10 minutes ago' --no-pager
journalctl --user -b -u xdg-desktop-portal.service --no-pager
```

| Filter | Meaning |
| --- | --- |
| `-b` | Current boot; `-b -1` selects the previous boot. |
| `-u NAME` | Messages associated with a unit. |
| `--user` | Query the current user's journal scope. |
| `-k` | Kernel messages. |
| `-p warning..alert` | Select a priority range. |
| `--since` / `--until` | Select a time interval. |
| `-f` | Follow new messages until interrupted. |

A warning is not automatically a fault, and a quiet journal is not proof of
correct behavior. Correlate timestamps, unit status, observable behavior, and
the current boot. Review logs before sharing them because they may contain
usernames, device identifiers, paths, network information, or application
data.

## Diagnosing a failed unit

Use a bounded sequence:

```bash
systemctl --failed --no-pager
systemctl status name.service --no-pager
journalctl -b -u name.service --no-pager
systemctl cat name.service
systemctl show -p LoadState -p ActiveState -p SubState -p Result name.service
```

Then identify:

1. Was the unit definition found and parsed?
2. Which job or process failed, with what result?
3. What pulled the unit into the transaction?
4. Is its application configuration valid?
5. Did an ordering dependency, missing device, mount, credential, permission,
   or network assumption fail?
6. Is the observed log from the relevant boot and scope?

After fixing the cause, start the unit again. `systemctl reset-failed name`
clears the recorded failed state and restart counters; it does not repair the
configuration or start the unit.

## Boot analysis

The final verification chapter uses:

```bash
systemd-analyze time
systemd-analyze critical-chain
```

`time` reports broad boot phases. `critical-chain` shows one time-critical
ordering chain based on recorded activation times. It is not a complete causal
profile: parallel jobs, device waits, services considered ready before their
work finishes, and later user-session startup can make a simple ranking
misleading.

Investigate a reproducible problem before optimizing. A service taking time is
not automatically unnecessary, and disabling units solely to improve a boot
score can remove security, maintenance, or recovery behavior.

## Editing and overriding units safely

The [configuration files and drop-ins guide](01-configuration-files-and-drop-ins.md)
explains precedence in detail. For a unit, create a local override with:

```bash
sudo systemctl edit name.service
sudo systemctl daemon-reload
systemctl cat name.service
```

Validate unit syntax and dependency issues before restart:

```bash
systemd-analyze verify name.service
```

When the unit exists only under `/usr`, pass its full path if verification
cannot resolve it as expected. Remember that directive semantics vary: some
assignments replace values, while list-like directives may need an empty
assignment before rebuilding the list.

Remove only the local override through the reviewed systemctl workflow or by
removing its exact file, then run `daemon-reload` and inspect the effective
unit again. Do not edit the package copy under `/usr`.

## Project policy map

| Unit | Canonical policy | Reason |
| --- | --- | --- |
| `NetworkManager.service` | Enabled and active | Sole network manager. |
| `systemd-timesyncd.service` | Enabled and active | Time synchronization baseline. |
| `firewalld.service` | Enabled and active | Host firewall manager. |
| `nftables.service` | Not enabled separately | Firewalld owns the nftables ruleset. |
| `sshd.service` | Disabled and inactive | SSH client installation must not expose a server. |
| `bluetooth.service` | Enabled when Bluetooth is part of the workstation | BlueZ system service. |
| `cups.socket` | Conditional | Printing is socket-activated only when needed. |
| `paccache.timer` | Enabled and active | Periodic conservative package-cache maintenance. |
| `fstrim.timer` | Enabled and active | Periodic TRIM through the complete storage path. |
| `fwupd-refresh.timer` | Enabled | Metadata refresh; firmware installation remains manual. |
| `reflector.timer` | Disabled | Mirror changes remain deliberate. |
| `tlp.service`, `tlp-pd.service` | Enabled and active as applicable | Selected power-management stack. |
| `passim.service` | Masked | Avoid the unwanted listener in this profile. |
| `greetd.service` | Enabled only after manual Niri and lock testing | Preserve TTY recovery during setup. |

This table records project intent, not universal Arch defaults. Hardware,
packages, and selected features determine which conditional units exist.

## Safe operating workflow

Before changing a unit:

1. Inspect `status`, `is-enabled`, `cat`, and relevant journal entries.
2. Determine what activates it and what depends on it.
3. Identify whether the change belongs in application configuration, a unit
   drop-in, an enablement link, or the source consumed by a generator.
4. Preserve a local TTY or other recovery path for networking, login, storage,
   and power changes.
5. Make one bounded change.
6. Run `daemon-reload` only when unit definitions or generator inputs require
   it.
7. Reload or restart the application only when its documented activation
   procedure requires it.
8. Recheck runtime state, enablement, listeners, behavior, and journal.

## Recovery principles

- A bad unit override can be removed from the local drop-in directory from a
  TTY or chroot, followed by `systemctl daemon-reload` on the running system.
- A service that repeatedly restarts should be stopped while diagnosing its
  application configuration and journal; clearing `failed` is not a fix.
- A broken greeter is disabled from another TTY so normal console login remains
  available.
- A network or firewall service is changed from a local console when losing
  connectivity would remove the only recovery route.
- Generated units are repaired by correcting `/etc/fstab`, crypttab, kernel
  command-line, zram-generator, or the relevant source—not `/run` output.
- Masking is reversed only after the original reason and dependent units are
  understood.

## Sources

- [systemd(1)](https://man.archlinux.org/man/systemd.1)
- [systemd.unit(5)](https://man.archlinux.org/man/systemd.unit.5)
- [systemd.service(5)](https://man.archlinux.org/man/systemd.service.5)
- [systemd.socket(5)](https://man.archlinux.org/man/systemd.socket.5)
- [systemd.timer(5)](https://man.archlinux.org/man/systemd.timer.5)
- [systemd.target(5)](https://man.archlinux.org/man/systemd.target.5)
- [systemd.generator(7)](https://man.archlinux.org/man/systemd.generator.7)
- [systemctl(1)](https://man.archlinux.org/man/systemctl.1)
- [journalctl(1)](https://man.archlinux.org/man/journalctl.1)
- [systemd-analyze(1)](https://man.archlinux.org/man/systemd-analyze.1)
- [ArchWiki: systemd](https://wiki.archlinux.org/title/Systemd)
- [ArchWiki: systemd/Timers](https://wiki.archlinux.org/title/Systemd/Timers)
- [ArchWiki: systemd/User](https://wiki.archlinux.org/title/Systemd/User)
