# Journal-led diagnosis, update incidents, and chroot recovery

## Purpose and scope

A recoverable system is not one that never fails. It is one whose operator can
identify the last working boundary, preserve evidence, choose the smallest
appropriate repair, and prove that the repair restored the intended state.

This guide turns the concepts from the first nineteen guides into one operating
method. It explains:

- how to distinguish routine maintenance from an incident;
- how to ask bounded questions of the system and user journals;
- how to correlate symptoms, timestamps, package transactions, units, kernel
  messages, and boot stages;
- how to respond to an interrupted or problematic full upgrade;
- when a TTY, fallback UKI, installation ISO, or `arch-chroot` is the correct
  recovery layer;
- how to inspect and rebuild this project's signed boot artifacts without
  recreating storage or trust state;
- how to maintain the workstation on a recurring, evidence-based cadence.

It does not add packages, automate upgrades, install another kernel, enable a
rollback framework, migrate ext4 to Btrfs, replace systemd-boot with GRUB,
repair a damaged filesystem automatically, or provide a universal command to
paste during every failure. Destructive storage, LUKS-header, Secure Boot-key,
and repository-repair operations still require a case-specific plan.

## Current project contract

All recovery examples assume the published profile, not older experimental
names:

| Layer | Canonical object |
| --- | --- |
| Computer | Lenovo ThinkPad T14 Gen 1 AMD |
| Firmware mode | UEFI with Secure Boot enabled |
| ESP | `/dev/nvme0n1p1`, FAT32, mounted at `/boot` |
| Encrypted partition | `/dev/nvme0n1p2`, LUKS2 |
| Open mapping | `/dev/mapper/cryptlvm` |
| Volume group | `vg0` |
| Root | `/dev/vg0/root`, ext4, mounted at `/` |
| Home | `/dev/vg0/home`, ext4, mounted at `/home` |
| Disk swap | `/dev/vg0/swap`, 16 GiB, inside LUKS |
| Normal UKI | `/boot/EFI/Linux/arch-linux.efi` |
| Fallback UKI | `/boot/EFI/Linux/arch-linux-fallback.efi` |
| Boot manager | Signed systemd-boot, normal and UEFI fallback paths |
| Secure Boot state | Owner keys and Microsoft certificates managed by `sbctl` |
| Kernel command line | Embedded in both UKIs from `/etc/kernel/cmdline` |
| Graphical session | greetd to `niri-session`, with TTY recovery retained |
| Package policy | Deliberate complete upgrades only; no automatic confirmation |
| Data recovery | Restic, raw encrypted recovery bundle, Git remotes, and verified Arch ISO |

The loader has `editor no`. The embedded kernel command line cannot be changed
interactively from the systemd-boot menu. That is a deliberate integrity and
reproducibility choice: a required command-line repair is made in the source
file and rebuilt into signed UKIs from a working system or chroot.

The fallback UKI uses the same installed kernel and command line as the normal
UKI. It contains a broader initramfs because it skips `autodetect`; it is not an
older kernel, package snapshot, or complete rollback.

## Maintenance and incidents are different modes

Routine maintenance begins with a working system and enough time to stop. An
incident begins with an unexplained symptom, failed transaction, degraded
service, lost boot boundary, or suspected data/storage problem.

| Mode | Objective | Appropriate behavior |
| --- | --- | --- |
| Routine maintenance | Keep known-good state current and recoverable | Read, update deliberately, verify, record |
| Investigation | Establish what failed and when | Preserve logs, narrow scope, avoid unrelated changes |
| Recovery | Restore one proven boundary | Apply the minimum reviewed repair and verify it |
| Improvement | Change design after stability returns | Use a separate commit, test, rollback plan, and documentation update |

Do not redesign the system inside an incident. A failed UKI is not the moment
to migrate boot managers; a filesystem error is not the moment to migrate to
Btrfs; a broken graphical session is not the moment to replace every desktop
component.

## The least-destructive recovery hierarchy

Escalate only when the lower layer cannot provide the required access:

1. **Application inspection:** the base system and session work; inspect the
   application, its files, and its logs.
2. **Graphical-session inspection:** use Kitty or another terminal while Niri
   still works.
3. **Local TTY:** use `Ctrl+Alt+F2` or another free console when the greeter,
   compositor, graphics, or user session fails.
4. **Fallback UKI:** use once when the normal early userspace may lack a
   required module.
5. **Arch ISO:** use when neither installed UKI supplies a usable system or an
   offline filesystem check is required.
6. **Mounted installation:** unlock LUKS, activate LVM, and inspect the
   installed files and journal from the ISO.
7. **`arch-chroot`:** enter only when installed commands, package management,
   UKI generation, signing, or configuration repair are required.
8. **Data or metadata restoration:** restore selected files, repository data,
   boot identity, or a LUKS header only after the failed object and target are
   proven.
9. **Reinstallation:** rebuild from the runbook after storage replacement or
   when a reviewed recovery decision determines repair is less reliable.

Being able to use a chroot does not mean a chroot is always needed. If a TTY
works, it provides the installed kernel, devices, network, journal, and service
manager directly. If the root filesystem may be damaged, mounting and entering
it before an offline check may be the wrong first action.

## The evidence-first incident loop

Use the same loop for packages, services, boot, storage, and the desktop.

### 1. Describe the observable symptom

Record what happened without explaining it prematurely:

- expected behavior;
- actual behavior and exact visible message;
- first known occurrence and last known-good occurrence;
- whether it is reproducible;
- whether it affects one application, one user, one boot, or the whole system;
- what changed immediately before it.

“Niri is broken” is not a bounded symptom. “After the 14:20 package
transaction, tuigreet accepts the password and returns to the greeter; TTY2
login works” identifies several successful boundaries and a narrow failure.

### 2. Preserve the current state

Before changing it:

```bash
date --iso-8601=seconds
uname -a
cat /etc/os-release
systemctl is-system-running
systemctl --failed --no-pager
systemctl --user --failed --no-pager
sudo journalctl --list-boots
sudo tail -n 200 /var/log/pacman.log
```

Not every command applies to every incident. The point is to retain boot,
kernel, system state, failed-unit, and recent package context before a reboot,
vacuum, reinstall, or configuration edit changes the evidence.

Logs can contain usernames, home paths, document names, device identifiers,
network addresses, SSIDs, URLs, application content, and authentication
details. Store incident notes below the private backed-up `System-Records`
directory, not in the public project repositories. Redact before sharing.

### 3. Identify the last completed boundary

Ask what is already proven:

- Did firmware find the loader?
- Did Secure Boot execute it?
- Did the menu list both UKIs?
- Did the selected UKI begin?
- Did the LUKS prompt appear and accept input?
- Did `vg0` activate and root mount?
- Did the normal system reach a TTY?
- Did greetd create a login session?
- Did `niri-session` start the user environment?
- Did only one application fail?

Start inspection immediately after the last confirmed boundary. Do not repair
earlier layers that visibly worked.

### 4. Narrow time, boot, unit, and subsystem

Prefer one precise query to a screenful of all history. Correlate it with the
real time of the symptom and the package log.

### 5. Form one falsifiable hypothesis

Examples:

- the kernel update completed, but the UKI hook failed because `/boot` was
  full;
- greetd works, but the Niri command or PAM stack fails;
- the unit file parses, but a missing mount prevents startup;
- the normal UKI omitted a module that the fallback contains;
- an application still runs with an old library because it was not restarted.

### 6. Make one bounded change

Preserve the previous file or Git state, validate syntax, and avoid changing a
second subsystem “while here”. A change without an isolated hypothesis makes a
later success or failure harder to explain.

### 7. Reproduce and verify at every relevant layer

A service becoming `active` does not prove its socket, firewall rule, portal,
or user-facing behavior. A generated UKI is not ready until it exists, can be
inspected, is discovered, and verifies as signed. A reboot is not successful
until the expected entry, mounts, services, journal, and application behavior
are checked.

### 8. Record cause, repair, and prevention

A useful private incident record contains:

```text
Date and boot ID:
Observable symptom:
Last known good state:
Recent changes:
Last completed boundary:
Evidence and exact log window:
Hypothesis tested:
Change made:
Verification performed:
Root cause or current uncertainty:
Follow-up documentation or automation:
```

Do not turn an unproven guess into project policy.

## Reading the journal as evidence

Guide 03 explains journal structure and systemd units. This section applies
those concepts to incidents.

### Select the correct boot first

```bash
sudo journalctl --list-boots
sudo journalctl -b --no-pager
sudo journalctl -b -1 --no-pager
```

`-b` means the current boot and `-b -1` the previous boot available in the
selected journal. Do not assume that previous-boot logs exist: they may have
been stored only in `/run`, rotated, vacuumed, or never flushed to persistent
storage. The list of boots is the evidence of what remains queryable.

Use an exact 32-character boot ID from `--list-boots` when offsets might be
ambiguous:

```bash
sudo journalctl --boot=BOOT_ID --no-pager
```

### Filter by severity without treating it as truth

```bash
sudo journalctl -b -p warning --no-pager
sudo journalctl -b -p err --no-pager
sudo journalctl -k -b -p warning --no-pager
```

A single priority selects that priority and every more severe level. Producers
choose their own priorities, so a warning can be harmless probing and a real
functional fault can be logged as informational text. An empty error query is
not proof of a healthy system; repeated correlated I/O, ext4, NVMe, amdgpu,
authentication, or suspend messages deserve investigation.

### Filter by unit and scope

```bash
systemctl status NAME.service --no-pager
sudo journalctl -b -u NAME.service --no-pager
systemctl --user status NAME.service --no-pager
journalctl --user -b -u NAME.service --no-pager
```

System and user managers are different scopes. `greetd.service`, firewalld,
NetworkManager, TLP, and system mounts belong to the system manager. Portals,
PipeWire, Mako, Waybar, and many graphical-session services belong to the user
manager. Querying the wrong scope can produce a convincing but irrelevant
empty result.

`systemctl status` shows current state and only a recent log excerpt. Use
`journalctl -u` for earlier invocations and the complete relevant window.

### Filter by time around a reproduced symptom

```bash
sudo journalctl --since '10 minutes ago' --no-pager
sudo journalctl --since '2026-08-31 14:20:00' --until '2026-08-31 14:25:00' -o short-full --no-pager
```

Replace the example dates with the recorded event. `short-full` prints a
locale-independent timestamp format that can be reused in `--since` and
`--until`.

For a safe reproducible event, follow one unit in another terminal:

```bash
sudo journalctl -f -u NAME.service
```

Stop following with `Ctrl+C`. Do not reproduce destructive storage, firmware,
credential, or package failures merely to obtain a cleaner log.

### Kernel, userspace, and package history complement one another

```bash
sudo journalctl -k -b --no-pager
sudo journalctl -b -u systemd-udevd.service --no-pager
sudo tail -n 200 /var/log/pacman.log
```

The kernel journal explains device discovery, drivers, I/O, filesystems, and
some suspend behavior. Unit logs explain supervised userspace. `pacman.log`
establishes package transaction time and versions. None alone proves cause.

Do not use `journalctl -x` when collecting logs for a bug report: catalog text
adds local explanatory material that is not part of the original event.

### Journal storage is finite evidence

```bash
sudo journalctl --disk-usage
sudo journalctl --verify
```

`--verify` checks journal-file internal consistency; without Forward Secure
Sealing keys it does not prove cryptographic authenticity. This project does
not run a routine manual vacuum. Vacuuming deletes older evidence and should
not be the first response to disk pressure or an active incident.

## A known-good health snapshot

Evidence is easier to interpret when a normal baseline exists. After a cold
boot known to be healthy, inspect:

```bash
hostnamectl
uname -r
timedatectl
systemctl is-system-running
systemctl --failed --no-pager
systemctl --user --failed --no-pager
sudo journalctl -b -p warning --no-pager
journalctl --user -b -p warning --no-pager
findmnt /
findmnt /home
findmnt /boot
df -hT / /home /boot
df -ih / /home /boot
swapon --show
sudo bootctl --esp-path=/boot status
sudo bootctl --esp-path=/boot list
sudo sbctl status
sudo sbctl verify
```

The baseline is not required to be silent. Firmware and drivers may emit known
warnings. Record why a recurring message is understood, the package and kernel
versions where it was observed, and which functional test proves it
non-blocking. Do not suppress it merely to produce clean output.

### Classify recurring platform messages before changing the system

This workstation profile can expose messages whose journal priority is more
alarming than their functional effect. Classify the exact text, the affected
subsystem, and a real hardware test separately.

The kernel line below is currently a probing artifact on systems without Intel
Trust Domain Extensions, including AMD machines:

```text
virt/tdx: TDX not supported by the host platform
```

An upstream kernel patch describes absence of that feature as a normal state
and removes the error-level message. On this AMD workstation, do not add a
kernel blacklist or command-line workaround solely to silence it. Record the
kernel version and re-evaluate after normal kernel updates.

These two lines have also been reported on the same ThinkPad generation:

```text
Serial bus multi instantiate pseudo device driver INT3515:00: error -ENXIO: IRQ index 1 not found
Serial bus multi instantiate pseudo device driver INT3515:00: error -ENXIO: Error requesting irq at index 1
```

That matching report does not prove the firmware cause or make every related
failure harmless. If the local text is identical, first test USB-C charging,
USB-C data, external display output, suspend, and resume, and keep firmware
current through the manual policy in post-install chapter 06. If those
functions work and the message occurs only during device discovery, record it
as a known platform warning for the tested kernel and firmware versions. Do
not suppress the driver message.

Wi-Fi messages require stricter correlation. `wlp3s0` and any associated P2P
interface can appear in the same boot, but a P2P setup failure is not evidence
that the primary connection failed, and a working connection is not evidence
that repeated authentication or roaming failures are irrelevant. Capture the
complete messages and both supervising layers:

```bash
sudo journalctl -b _COMM=wpa_supplicant --no-pager
sudo journalctl -b -u NetworkManager.service -u wpa_supplicant.service --no-pager
nmcli general status
nmcli device status
lspci -nnk | grep -A3 -i 'network'
```

Then compare their timestamps with an observed disconnect, failed association,
missing network, or successful continuous connection. Do not change the Wi-Fi
backend, disable P2P, or mask a unit from an isolated priority-filtered line.

## The recurring maintenance cadence

Arch requires regular operator attention, not unattended upgrades. The cadence
below is risk-based rather than a promise that every command must run on one
fixed calendar day.

### Before every complete upgrade

1. Ensure the Arch ISO, LUKS passphrase, Restic password, and recovery route are
   available.
2. Avoid the transaction immediately before class, travel, a deadline, or
   other critical work.
3. Read every Arch News entry newer than the last successful upgrade.
4. Inspect pending official updates without altering the live sync database:

   ```bash
   checkupdates
   ```

5. Check root, home, and ESP capacity:

   ```bash
   df -hT / /home /boot
   df -ih / /home /boot
   ```

6. Commit and push reviewed repository work; do not hide unrelated dirty files
   inside an update-related commit.
7. Run a current Restic backup before a high-risk update or substantial local
   change, and verify its result as guide 19 describes.
8. Confirm enough battery or AC power and enough time for package hooks and a
   controlled reboot.

`checkupdates` is informational. No output with exit status `2` means no
updates are available. It neither replaces Arch News nor authorizes partial
package installation.

`checkupdates` is shipped by `pacman-contrib`, but its isolated database
operation also uses `fakeroot`. Arch records that helper as an optional
dependency because the same package contains tools that do not need it;
optional dependencies are reported during installation but are not selected
automatically. Verify both parts when the command reports that it cannot find
the fakeroot binary:

```bash
pacman -Q pacman-contrib fakeroot
```

The `base-devel` metapackage also brings in `fakeroot`, along with the complete
standard Arch package-building toolchain. Installing the metapackage is correct
when building repository or AUR packages. Installing only `fakeroot` is the
smaller and clearer fix when the sole requirement is `checkupdates`.

### During the upgrade

Use the supported transaction:

```bash
sudo pacman -Syu
```

Read:

- the complete package list, sizes, replacements, conflicts, and provider
  choices;
- package messages requiring manual intervention;
- `.pacnew`, `.pacsave`, and `.pacorig` notices;
- kernel, mkinitcpio, UKI, signing, boot-loader, cache, schema, and daemon hooks;
- every warning or error before closing the terminal.

Never use isolated `pacman -Sy`, `--noconfirm`, `--overwrite`, `--nodeps`, or
`-Rdd` to force an unexplained transaction through.

### Before rebooting after an upgrade

```bash
sudo pacdiff --output
sudo tail -n 200 /var/log/pacman.log
systemctl --failed --no-pager
df -hT / /home /boot
sudo bootctl --esp-path=/boot list
sudo sbctl verify
```

Resolve each `.pacnew` through comparison and validation. If kernel,
microcode, systemd, mkinitcpio, systemd-ukify, systemd-boot, or sbctl changed,
also inspect:

```bash
ls -lh --time-style=long-iso /boot/EFI/Linux
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux-fallback.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
```

Both images must exist, identify as UKIs, contain the intended command line,
appear in `bootctl list`, and verify as signed. An unsigned
`/boot/vmlinuz-linux` remains expected because it is the package-managed input,
not a directly executed boot entry.

Do not reboot to “see whether it works” after a failed boot hook while the
running kernel still supplies a repair shell.

### After the controlled reboot

```bash
uname -r
systemctl is-system-running
systemctl --failed --no-pager
systemctl --user --failed --no-pager
sudo journalctl -b -p err --no-pager
sudo bootctl --esp-path=/boot status
sudo sbctl status
sudo sbctl verify
```

Then verify the subsystems affected by the transaction: network, firewall,
audio, Bluetooth, Niri, portals, power, suspend, removable media, or
applications. The package manager proving that files were installed does not
prove that long-running processes restarted or the user-facing behavior works.

### Weekly operational review

The exact weekday is unimportant. Review the supplied timers and failed state:

```bash
systemctl list-timers --all paccache.timer fstrim.timer fwupd-refresh.timer --no-pager
systemctl --failed --no-pager
systemctl --user --failed --no-pager
sudo pacdiff --output
```

Interpret timers rather than expecting their services to stay active:

- `paccache.timer` retains the conservative package-cache history;
- `fstrim.timer` sends periodic discard through ext4, LVM, and LUKS;
- `fwupd-refresh.timer` refreshes metadata but does not install firmware;
- `reflector.timer` remains disabled;
- no timer approves or installs system package upgrades.

### Monthly health review

```bash
df -hT / /home /boot
df -ih / /home /boot
sudo journalctl --disk-usage
sudo journalctl -p warning --since '1 month ago' --no-pager
pacman -Qdt
pacman -Qm
sudo paccache --dryrun
sudo bootctl --esp-path=/boot list
sudo sbctl verify
```

`pacman -Qdt` reports orphan candidates and `pacman -Qm` foreign packages.
Neither output authorizes automatic removal. A foreign package may require a
rebuild after official libraries change; an orphan may still be intentionally
used.

Use targeted ownership and package-file checks when a real symptom exists:

```bash
pacman -Qo /absolute/suspect/path
pacman -Qk PACKAGE
pacman -Qkk PACKAGE
```

Modified configuration and generated files can make `-Qkk` report deliberate
differences. It is evidence to interpret, not an instruction to reinstall the
whole system.

### Quarterly and after major architecture changes

- verify the current Restic snapshot, rotating data-read coverage, and a real
  staged restore;
- verify the raw recovery bundle and its second encrypted copy;
- boot the current verified Arch ISO and rehearse unlock, LVM activation,
  mounts, and chroot without repair;
- boot the fallback UKI once and return to the normal entry;
- review the current package and enabled-unit inventories;
- refresh the recovery bundle after LUKS, storage, Secure Boot, UKI, or
  material boot configuration changes;
- test a complete shutdown, cold boot, lock, repeated suspend/resume, and
  network reconnection;
- record the result in the private system record.

Firmware installation remains manual and separate from the ordinary pacman
transaction. Review power, vendor instructions, `fwupdmgr get-updates`, and
the boot/trust recovery path before approving one.

## When an upgrade reports a failure

### Do not close or reboot reflexively

If the terminal and running system still work:

1. preserve the complete visible error and relevant `pacman.log` window;
2. determine whether package download, validation, file installation, a
   scriptlet, or a post-transaction hook failed;
3. check root and ESP capacity;
4. keep the current running system available while repairing the exact cause;
5. complete the full transaction before installing unrelated packages.

A hook failure can occur after package files and the local database changed.
“Pacman returned nonzero” does not mean nothing was installed.

### A database lock is a coordination signal

Pacman creates `/var/lib/pacman/db.lck` to exclude another writer. Inspect
before removing it:

```bash
ps -ef | grep -E '[p]acman|[m]akepkg'
```

Only when every real package operation has ended and the previous process is
known to have terminated abnormally may a stale lock be removed:

```bash
sudo rm /var/lib/pacman/db.lck
```

The removal repairs only coordination. It does not finish the interrupted
transaction or prove package-database consistency. Inspect the log and resume
the complete upgrade.

### A partial-upgrade state must be completed

If `pacman -Sy` was run or a synchronized transaction was left incomplete, do
not install individual packages or create compatibility symlinks for missing
library SONAMEs. With a functioning installed pacman and network:

```bash
sudo pacman -Syu
```

Read and resolve the resulting transaction. Reboot afterward when the kernel
or low-level libraries changed.

### A failed UKI or signing hook blocks reboot

Inspect the source configuration and available space:

```bash
df -hT /boot
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/mkinitcpio.d/linux.preset
cat /etc/kernel/cmdline
ls -lh /boot/EFI/Linux
sudo sbctl verify
```

If configuration and space are correct and the package hook clearly failed,
rebuild both documented presets:

```bash
sudo mkinitcpio -P
```

Read every build and post-hook result. If required saved EFI artifacts remain
unsigned after successful generation:

```bash
sudo sbctl sign-all
sudo sbctl verify
```

Then repeat `bootctl list`, `kernel-inspect`, file-size, timestamp, and
signature checks. Do not treat `sign-all` as a substitute for fixing a UKI that
failed to build.

### Disk pressure is not permission for indiscriminate deletion

```bash
df -hT / /home /boot
df -ih / /home /boot
sudo du -xhd1 /var | sort -h
sudo paccache --dryrun
sudo journalctl --disk-usage
```

Determine whether bytes or inodes are exhausted and which filesystem is full.
The root filesystem's 1% reserve provides root with limited recovery space;
it is not capacity planning. Preserve logs during an incident. Never delete
current UKIs, `/boot/vmlinuz-linux`, pacman's local database, sbctl state, or
unidentified files merely to let a transaction continue.

On the ESP, list and identify every file before considering an obsolete copy:

```bash
find /boot/EFI -maxdepth 3 -type f -printf '%s %TY-%Tm-%Td %TH:%TM %p\n' | sort -n
sudo bootctl --esp-path=/boot list
sudo sbctl verify
```

Copy questionable artifacts to verified external storage before a reviewed
removal. The normal and fallback UKIs plus normal and UEFI fallback loader
paths are all intentional.

## Problematic upgrades and downgrade boundaries

After a successful full transaction, first rule out configuration, stale
processes, and known manual intervention:

1. read relevant Arch News and the package transaction messages;
2. inspect `/var/log/pacman.log` for exact old and new versions;
3. resolve `.pacnew` files;
4. restart the affected process or perform a controlled reboot;
5. inspect the relevant current and previous boot journals;
6. confirm whether the issue is already reported and fixed upstream;
7. reproduce on the second non-critical ThinkPad first when downtime matters.

The retained package cache can support a reviewed downgrade:

```bash
ls -1 /var/cache/pacman/pkg/PACKAGE-*.pkg.tar.*
sudo pacman -U /var/cache/pacman/pkg/EXACT_PACKAGE_FILE.pkg.tar.zst
```

The second command is a pattern showing the operation; use an exact inspected
filename, not the literal placeholder. A single-package downgrade can create a
partial system when libraries, dependencies, kernel modules, or related
packages changed together. It is a last-resort diagnostic or temporary
recovery step with a tracked route back to a complete supported state, not the
routine rollback model.

Do not create arbitrary library symlinks, permanently add `IgnorePkg`, or use
the Arch Linux Archive to assemble an unexplained mix of dates. When a coherent
historical repository state is genuinely required, plan that complete state
and recovery route separately.

## Recovering the graphical system from a TTY

If the base system reaches a TTY, boot, LUKS, LVM, root, systemd, and login are
already working. Keep the diagnosis above the proven boundary.

Log in on another console and inspect:

```bash
systemctl status greetd.service --no-pager
sudo journalctl -b -u greetd.service --no-pager
systemctl --user --failed --no-pager
journalctl --user -b -p warning --no-pager
niri validate
```

For a failing user component:

```bash
systemctl --user status NAME.service --no-pager
journalctl --user -b -u NAME.service --no-pager
systemctl --user cat NAME.service
```

Check the exact session command, PAM messages, executable availability, Stow
links, environment propagation, and user-unit scope. A broken Mako, Waybar,
portal, or application does not justify rebuilding UKIs or entering a chroot.

If greetd repeatedly prevents useful graphical login, stop it temporarily from
the TTY while retaining the console:

```bash
sudo systemctl stop greetd.service
```

Repair and validate the cause before starting it again. Do not disable or mask
it permanently as an unexplained workaround.

## Diagnose boot by the last successful stage

| Symptom | Proven stage | Inspect next | Do not do first |
| --- | --- | --- | --- |
| No Linux Boot Manager entry | Firmware runs | NVRAM order, ESP identity, loader files, UEFI fallback path | Repartition the ESP |
| Security violation before menu | Firmware found an EFI object | Enrollment and systemd-boot signature | Clear keys or enter Setup Mode |
| Menu opens but one UKI is rejected | Loader executed | Selected UKI signature and file integrity | Regenerate all Secure Boot keys |
| UKI starts but no LUKS prompt | Kernel/early userspace started | Embedded command line, initramfs hooks, NVMe driver, fallback UKI | Restore LUKS header |
| Prompt appears but passphrase fails | LUKS device was found | Keyboard layout, correct passphrase, keyslots | `luksFormat` or immediate header restore |
| LUKS opens but root is absent | Decryption succeeded | `lvm2` hook, `vg0`, LV state, `root=` path | `pvcreate` or `vgcreate` |
| Root exists but cannot mount | Storage through LVM succeeded | ext4 journal, filesystem state, space, fsck result | Format the LV |
| Emergency mode after root mount | Switch-root likely succeeded | `/etc/fstab`, generated mounts, failed units, journal | Reinstall the boot loader |
| TTY works but graphical login fails | Complete base boot succeeded | greetd, PAM, Niri, user manager, graphics | Reinstall Arch |
| One application fails | Desktop and session work | Its package, config, data, portal, and logs | Change boot or storage layers |

This table is a boundary map, not an automatic repair table.

## Entering the installed system from the Arch ISO

### Prepare safely

Use a current official ISO whose checksum or PGP signature was verified. Keep
the recovery bundle disconnected until its contents are needed. If custom
Secure Boot rejects the ISO, temporarily disable Secure Boot; do not clear
owner keys or enable Setup Mode.

Boot in UEFI mode and identify the actual internal disk:

```bash
cat /sys/firmware/efi/fw_platform_size
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,TRAN
```

The first command must print `64`. Stop if the disk layout does not match the
machine being recovered.

### Reconstruct existing layers without creating them

```bash
cryptsetup open /dev/nvme0n1p2 cryptlvm
cryptsetup status cryptlvm
vgchange --activate y vg0
pvs
vgs
lvs -a -o +devices
```

These commands open and activate existing metadata. Do not substitute
`luksFormat`, `pvcreate`, `vgcreate`, `lvcreate`, or filesystem creation
commands.

If the incident suggests ext4 damage, stop before mounting and follow the
offline filesystem section below. Otherwise mount the canonical hierarchy:

```bash
mount /dev/vg0/root /mnt
mount --mkdir /dev/vg0/home /mnt/home
mount --mkdir -o fmask=0177,dmask=0077 /dev/nvme0n1p1 /mnt/boot
findmnt /mnt /mnt/home /mnt/boot
```

Verify every source. Mounting the ESP at `/mnt/boot` is necessary because UKI
generation and bootctl must operate on the real boot partition, not an empty
directory on root.

### Read the failed installed boot before entering chroot

The ISO has its own current journal. Query the mounted installation explicitly:

```bash
journalctl --root=/mnt --list-boots
journalctl --root=/mnt --boot=INSTALLED_BOOT_ID -p warning --no-pager
```

Select the exact ID printed for the installed boot. Do not use `-b -1`
blindly: relative offsets are evaluated within the selected journal set, and
the failed installed boot may be its most recent entry.

Also inspect package and source configuration directly:

```bash
tail -n 200 /mnt/var/log/pacman.log
cat /mnt/etc/fstab
cat /mnt/etc/kernel/cmdline
grep '^HOOKS=' /mnt/etc/mkinitcpio.conf
cat /mnt/etc/mkinitcpio.d/linux.preset
ls -lh /mnt/boot/EFI/Linux
```

### Enter only when installed tools are required

```bash
arch-chroot /mnt
```

`arch-chroot` arranges `/dev`, `/proc`, `/sys`, and resolver access around the
new root. It is not a boot: the ISO kernel is still running, ordinary services
are not started as if the installed system had booted, and D-Bus-dependent
tools can fail. `systemctl start` inside a conventional chroot is not a valid
test of the eventual boot.

Inside the chroot, establish the state before writing:

```bash
findmnt /
findmnt /home
findmnt /boot
df -hT / /home /boot
ls -l /usr/lib/modules
pacman -Q linux systemd mkinitcpio systemd-ukify sbctl
tail -n 200 /var/log/pacman.log
bootctl --esp-path=/boot list
sbctl status
sbctl verify
```

Firmware state reported from a chroot can be limited by how efivars and the
namespace are exposed. File, UKI, configuration, package, and signature
inspection remain useful even when NVRAM mutation is unavailable.

## Repair classes inside the chroot

### Complete an interrupted package transaction

If the installed `pacman` runs, the network is available, and the evidence
shows an incomplete full upgrade:

```bash
pacman -Syu
```

Use `pacman.log` to identify packages whose installation or hooks were
interrupted. Reinstall an exact affected set only when the ordinary completion
does not rerun the required scriptlets or hooks:

```bash
grep '\[ALPM\] upgraded' /var/log/pacman.log | tail -n 100
pacman -Syu PACKAGE1 PACKAGE2
```

Replace the placeholders with the inspected packages from the failed
transaction. Do not paste a date-dependent shell pipeline that automatically
constructs a write transaction without first reviewing its output.

If the installed pacman cannot execute because its own libraries are broken,
exit the chroot. The ISO pacman can use the installed root as a sysroot:

```bash
exit
pacman -Syu --sysroot /mnt
arch-chroot /mnt
```

This is a fallback for a broken installed package manager, not the preferred
routine update method. Preserve the log and confirm that the command uses the
target's configuration, database, keyring, mirrors, and complete upgrade
boundary.

### Rebuild the existing UKIs

Only after the inputs and ESP have been verified:

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
cat /etc/mkinitcpio.d/linux.preset
df -hT /boot
mkinitcpio -P
```

`-P` processes every preset in `/etc/mkinitcpio.d`; in this project the Linux
preset builds the normal and fallback UKIs. Read every warning and confirm that
both builds and their post-hooks finish.

Inspect the resulting objects:

```bash
ls -lh /boot/EFI/Linux
bootctl kernel-identify /boot/EFI/Linux/arch-linux.efi
bootctl kernel-identify /boot/EFI/Linux/arch-linux-fallback.efi
bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
bootctl --esp-path=/boot list
sbctl verify
```

If the builds succeeded but required saved EFI files remain unsigned:

```bash
sbctl sign-all
sbctl verify
```

Do not regenerate keys. `/var/lib/sbctl` contains the established owner
identity needed to repair signatures.

### Repair systemd-boot only when it is the failed object

First inspect normal and fallback loader files plus the signed package source:

```bash
ls -lh /boot/EFI/systemd/systemd-bootx64.efi
ls -lh /boot/EFI/BOOT/BOOTX64.EFI
ls -lh /usr/lib/systemd/boot/efi/systemd-bootx64.efi*
bootctl --esp-path=/boot status
sbctl verify
```

If diagnosis proves that the ESP copies are missing or stale while the saved
signed source is valid, `bootctl --esp-path=/boot update` can refresh existing
systemd-boot installations. Do not run `install` or recreate the firmware
entry merely because a UKI failed later in the chain.

After a deliberate loader update, repeat `sbctl verify` and inspect both ESP
paths. A normal chroot may not update UEFI variables; the standard
`/EFI/BOOT/BOOTX64.EFI` fallback path can still provide a firmware-independent
route while the NVRAM entry is repaired separately.

### Repair generated mounts at their source

If emergency mode names a generated mount or swap unit, inspect:

```bash
cat /etc/fstab
findmnt --verify --verbose
cat /etc/systemd/zram-generator.conf
```

Correct `/etc/fstab`, the relevant generator configuration, or the real
device identity. Do not edit generated files under `/run/systemd/generator`;
they disappear and are recreated from their sources.

## Offline ext4 diagnosis

Do not run `e2fsck` on mounted root or home. Even a nominal read-only check can
produce invalid results against a changing mounted filesystem.

From the ISO, open LUKS and activate `vg0`, but leave the target LV unmounted.
Prove that first:

```bash
findmnt /dev/vg0/root
findmnt /dev/vg0/home
```

No output is expected for the LV being checked. Preserve backups and inspect
hardware/I/O evidence before attributing corruption to ext4. A non-writing,
forced diagnostic check can be performed on one exact unmounted LV:

```bash
e2fsck -fn /dev/vg0/root
```

`-n` answers no to repairs and `-f` forces a full check. Use
`/dev/vg0/home` only when that is the verified target. This is evidence, not
repair.

An interactive repair can change filesystem metadata and may place recovered
objects in `lost+found`. Never answer every prompt automatically with `-y`
without a current backup, stable hardware, recorded output, and a
case-specific recovery decision. If the NVMe or transport reports I/O errors,
preserve the source and prioritize hardware/data recovery over repeated fsck
writes.

Ext4 normally replays its journal after an unclean shutdown and marks the
filesystem clean when no further check is needed. A forced offline full scan is
not monthly preventive maintenance for an otherwise healthy mounted
filesystem.

## Leave the recovery environment cleanly

Inside the chroot, perform final read-only verification and then leave:

```bash
bootctl --esp-path=/boot list
sbctl verify
exit
```

From the ISO:

```bash
sync
umount -R /mnt
vgchange --activate n vg0
cryptsetup close cryptlvm
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

If unmount reports a busy target, find the shell or process still using
`/mnt`; do not use forced or lazy unmount as routine recovery. Confirm that the
mapping is closed before disconnecting media or rebooting.

If Secure Boot was temporarily disabled only to start the ISO, re-enable it
without clearing or replacing keys. Boot the normal UKI first, then verify the
active loader, command line, storage map, failed units, journal, Secure Boot
state, and the subsystem that originally failed. Test the fallback at a later
controlled point if it was rebuilt.

## Common incident playbooks

### Secure Boot violation before the menu

1. Do not clear keys or recreate sbctl state.
2. Temporarily disable Secure Boot and try the existing normal and UEFI
   fallback loader paths.
3. Boot the ISO and mount the installation.
4. Inspect `sbctl status`, `sbctl verify`, both systemd-boot files, both UKIs,
   and the package transaction that last touched them.
5. Rebuild or re-sign only the proven failed object.
6. Verify every required signature before re-enabling enforcement.

### Normal UKI fails but fallback reaches the system

1. Treat fallback success as evidence that firmware, loader, kernel family,
   LUKS metadata, LVM, and root remain broadly usable.
2. Compare `kernel-inspect` output, file sizes, timestamps, and build logs.
3. Inspect why `autodetect` omitted a required early module.
4. Correct the mkinitcpio source and rebuild both images.
5. Do not keep the broader fallback permanently as an unexplained default.

If both fail after the same kernel update, remember that they use the same
kernel. The fallback is not an older-kernel escape path.

### LUKS opens but `vg0` or root is missing

1. Confirm `cryptsetup status cryptlvm` and the backing device.
2. Inspect `pvs`, `vgs`, and `lvs -a -o +devices` from the ISO.
3. Check the embedded `root=/dev/mapper/vg0-root` and `lvm2` initramfs hook.
4. Preserve LVM metadata and logs.
5. Never use LVM creation commands to rediscover an existing VG.

### Root mounts but boot enters emergency mode

1. Read the exact failed unit and current boot journal.
2. Inspect `systemctl --failed`, generated mount units, `/etc/fstab`, and
   `findmnt --verify --verbose`.
3. Correct the source record, not `/run` output.
4. Use `systemctl daemon-reload` and retry the exact mount only on the running
   system where that test is safe.
5. Reboot only after the corrected dependency and mount path verify.

### Update was interrupted and the system no longer boots

1. Boot the ISO, identify the disk, open LUKS, activate LVM, and mount all
   three filesystems.
2. Read the installed journal and `/var/log/pacman.log` before entering chroot.
3. Complete the full upgrade with the installed pacman when possible.
4. Reinstall only the exact interrupted package set when evidence requires its
   scriptlets or hooks to rerun.
5. Rebuild and inspect UKIs if boot-related packages were involved.
6. Verify signatures and systemd-boot discovery.
7. Leave the storage stack cleanly and perform a controlled normal boot.

### A package or configuration file is missing

```bash
pacman -Qo /absolute/missing/or/damaged/path
pacman -Qk PACKAGE
pacman -Qkk PACKAGE
```

Use ownership to identify the authoritative package. Reinstalling its current
official version can restore package-owned files, but package backups and
administrator configuration under `/etc` may intentionally survive. Compare
`.pacnew`, package defaults, Git-tracked local policy, and the backup before
overwriting configuration.

## What not to automate

This project automates bounded maintenance whose policy is already understood:
package-cache retention, TRIM, and firmware-metadata refresh. It does not
automate:

- `pacman -Syu` or confirmation of package transactions;
- `.pacnew` merging;
- package downgrade or removal;
- journal vacuum during incidents;
- Restic retention or prune;
- firmware installation;
- `e2fsck` repair;
- LUKS header restoration;
- Secure Boot key enrollment or replacement;
- UKI or boot-loader repair triggered only by a generic error;
- automatic reboot after an upgrade.

Automation is appropriate only after inputs, success criteria, failure
reporting, recovery, concurrency, power, suspend, and credential behavior are
defined. A job that runs silently is not a maintenance policy.

## Decisions recorded for this project

- Ext4, LUKS2 over LVM, systemd-boot, mkinitcpio/ukify, signed UKIs, and sbctl
  remain the canonical architecture; incidents do not justify opportunistic
  migration.
- `cryptlvm` and `vg0` are the current names. Older `MyVolGroup` examples are
  not valid for this installation.
- Recovery begins at the last proven boundary and escalates from application
  to TTY, fallback, ISO, mounted inspection, and chroot.
- The fallback UKI provides broader modules, not an older kernel.
- Package upgrades remain complete, manual, and preceded by Arch News.
- A failed package hook can leave changed package state; its result is read
  before reboot.
- Pacman locks, journal warnings, failed units, and fsck output are evidence to
  interpret, not permission for generic cleanup.
- Both UKIs, systemd-boot discovery, embedded command lines, and Secure Boot
  signatures are verified before reboot after boot-chain changes.
- Downgrades are temporary, reviewed last resorts and do not become a frozen
  partial system.
- Chroot repair operates on the existing storage and trust layers; it does not
  recreate them.
- Ext4 checks run against one exact unmounted LV. Routine forced fsck is not
  part of the maintenance calendar.
- Logs and incident records remain private, backed-up data and are redacted
  before sharing.
- Upgrades, firmware, filesystem repair, boot-key changes, Restic pruning, and
  reboots remain operator-approved actions.

## Essential-edition operating checklist

- [ ] The canonical storage, boot, trust, session, and backup map is recorded.
- [ ] A known-good cold-boot baseline exists with explained recurring warnings.
- [ ] Arch News is read before every complete manual upgrade.
- [ ] Root, home, ESP, and inode capacity are checked before a large update.
- [ ] Pacman output, hooks, `.pacnew`, and `pacman.log` are reviewed.
- [ ] Boot-related updates leave two inspectable, discovered, signed UKIs.
- [ ] Failed system and user units plus the relevant journal window are checked
      after reboot.
- [ ] Weekly timers are inspected and no automatic package upgrade exists.
- [ ] Monthly space, journal, orphan, foreign-package, cache, boot, and
      signature state is reviewed without automatic deletion.
- [ ] Quarterly Restic, raw bundle, ISO/chroot, fallback, cold-boot, and
      suspend recovery tests are recorded.
- [ ] Every incident records the symptom, last completed boundary, evidence,
      one bounded change, and verification.
- [ ] TTY recovery works independently of greetd and Niri.
- [ ] The ISO can open `cryptlvm`, activate `vg0`, mount the canonical tree,
      read the installed journal, and enter `arch-chroot`.
- [ ] No recovery rehearsal formats storage, restores a LUKS header, replaces
      Secure Boot keys, or repairs ext4 merely as a test.
- [ ] Private logs, credentials, identifiers, and recovery material remain out
      of Git.

## Sources

- [Arch Linux News](https://archlinux.org/news/)
- [ArchWiki: System maintenance](https://wiki.archlinux.org/title/System_maintenance)
- [ArchWiki: General troubleshooting](https://wiki.archlinux.org/title/General_troubleshooting)
- [ArchWiki: Pacman](https://wiki.archlinux.org/title/Pacman)
- [ArchWiki: Chroot](https://wiki.archlinux.org/title/Chroot)
- [ArchWiki: mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio)
- [ArchWiki: systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)
- [`journalctl(1)`](https://man.archlinux.org/man/journalctl.1.en)
- [`systemctl(1)`](https://man.archlinux.org/man/systemctl.1.en)
- [`systemd(1)`](https://man.archlinux.org/man/systemd.1.en)
- [`pacman(8)`](https://man.archlinux.org/man/pacman.8)
- [`pacdiff(8)`](https://man.archlinux.org/man/pacdiff.8.en)
- [`checkupdates(8)`](https://man.archlinux.org/man/checkupdates.8.en)
- [Arch package: pacman-contrib](https://archlinux.org/packages/extra/x86_64/pacman-contrib/)
- [Arch package: fakeroot](https://archlinux.org/packages/core/x86_64/fakeroot/)
- [Linux kernel patch: do not print a TDX error when the feature is absent](https://lkml.rescloud.iu.edu/hypermail/linux/kernel/2607.3/10104.html)
- [Arch forum hardware report: INT3515 on ThinkPad T14s Gen 1 AMD](https://bbs.archlinux.org/viewtopic.php?id=293342)
- [`paccache(8)`](https://man.archlinux.org/man/paccache.8.en)
- [`arch-chroot(8)`](https://man.archlinux.org/man/arch-chroot.8.en)
- [`mkinitcpio(8)`](https://man.archlinux.org/man/mkinitcpio.8.en)
- [`bootctl(1)`](https://man.archlinux.org/man/bootctl.1.en)
- [`sbctl(8)`](https://man.archlinux.org/man/sbctl.8.en)
- [`cryptsetup-open(8)`](https://man.archlinux.org/man/cryptsetup-open.8.en)
- [`vgchange(8)`](https://man.archlinux.org/man/vgchange.8.en)
- [`findmnt(8)`](https://man.archlinux.org/man/findmnt.8.en)
- [`e2fsck(8)`](https://man.archlinux.org/man/e2fsck.8.en)

## Milestone reached

Guides 01 through 20 now form the first essential edition of the handbook.
Together they explain the configuration, package, service, boot, trust,
storage, network, workstation, Niri, backup, maintenance, diagnosis, and
recovery boundaries needed to operate the canonical ThinkPad installation.

The next accepted guide begins the extension series with shell and terminal
fundamentals. Those guides deepen skills and add optional capabilities; the
installed workstation no longer depends on them to be understandable or
recoverable.
