# From UEFI firmware to the mounted root filesystem

## Purpose and scope

The boot screen presents one continuous sequence, but several independent
environments cooperate before the normal system exists. Understanding their
boundaries makes failures diagnosable and explains why an encrypted machine
still needs readable files on an unencrypted EFI System Partition.

This guide traces the canonical project path:

1. UEFI firmware initializes the platform.
2. Secure Boot verifies systemd-boot.
3. systemd-boot discovers and selects a signed unified kernel image.
4. The UKI supplies the kernel, microcode, initramfs, metadata, and command
   line as one EFI executable.
5. The kernel enters the systemd-based initramfs.
6. Early userspace unlocks LUKS2, activates LVM, checks ext4, and mounts root.
7. Control switches to systemd in the installed root filesystem.
8. The normal system mounts `/home` and `/boot`, enables swap, starts services,
   and reaches the login target.

The guide explains architecture and diagnosis. The runbook remains the source
of exact installation commands, machine identifiers, and destructive storage
operations. A later guide will examine Secure Boot keys and threat boundaries
in greater depth.

## The canonical layers

| Layer | Canonical project object | Main responsibility |
| --- | --- | --- |
| Firmware | ThinkPad UEFI | Initialize hardware, expose UEFI services, select an EFI executable, enforce Secure Boot policy. |
| Boot partition | 1 GiB FAT32 ESP mounted at `/boot` | Store EFI executables, UKIs, loader configuration, and random seed. |
| Boot manager | Signed `systemd-bootx64.efi` | Discover entries, present the menu, select a UKI. |
| Boot artifact | Signed normal or fallback UKI | Bind boot-critical code and configuration into one executable. |
| Early kernel environment | systemd-based initramfs | Find storage, prompt for LUKS, activate LVM, check and mount root. |
| Encrypted block layer | LUKS2 mapping `cryptlvm` | Provide decrypted block access after successful unlock. |
| Volume layer | LVM volume group `vg0` | Expose `root`, `home`, and `swap` logical volumes. |
| Root filesystem | ext4 on `vg0/root` | Supply the installed operating system. |
| Normal userspace | systemd as PID 1 | Start the complete machine and user-session lifecycle. |

Each layer can succeed while the next fails. Reaching the systemd-boot menu
proves neither that LUKS can unlock nor that the root filesystem is healthy.
Reaching the LUKS prompt proves that firmware, boot manager, UKI, kernel, and
enough initramfs code already worked.

## Discovery and trust are separate chains

The firmware must both *find* an EFI executable and decide whether it may run.

The discovery path is:

```text
UEFI BootOrder → Linux Boot Manager entry → EFI/systemd/systemd-bootx64.efi
```

If the NVRAM entry is unavailable, UEFI defines a removable-media fallback
path for x86-64:

```text
EFI/BOOT/BOOTX64.EFI
```

The runbook installs signed copies of systemd-boot at both paths. This second
path is a firmware discovery fallback. It is not the `arch-linux-fallback.efi`
UKI and does not contain a different Linux kernel.

The trust path is conceptually:

```text
Firmware trust database → systemd-boot signature → selected UKI signature
```

Discovery answers “which file should be attempted?” Secure Boot answers “is
this EFI image signed by a key accepted by current firmware policy?” A correct
NVRAM path can point to an untrusted image, and a correctly signed image can be
unreachable because the boot entry or ESP is wrong.

Inspect firmware entries and order from Linux:

```bash
sudo efibootmgr -v
sudo bootctl status
sudo bootctl --print-loader-path
```

Do not delete unfamiliar EFI entries merely because they are old or duplicated.
An entry may belong to recovery media, Windows, another disk, or an earlier
installation. Identify its disk, partition GUID, and file path first.

### Why two entries can have the same name

`Linux Boot Manager` is only a human-readable label stored in a UEFI NVRAM
variable. It is not a unique identifier. Each `Boot####` variable also records
a device path similar to:

```text
HD(1,GPT,<ESP-PARTUUID>,...)/File(\EFI\systemd\systemd-bootx64.efi)
```

Repartitioning a disk creates a new GPT partition GUID even when the new ESP
uses the same disk, partition number, mount point, label, and loader path as the
old one. Firmware can therefore retain an obsolete `Linux Boot Manager` entry
for the previous PARTUUID while `bootctl install` or `efibootmgr --create`
adds a second entry with the same visible label for the new ESP.

The identifiers involved are not interchangeable:

| Identifier | Names | Used for |
| --- | --- | --- |
| GPT partition GUID | `PARTUUID`, GUID inside `HD(...,GPT,...)` | Let a UEFI boot variable identify the ESP partition. |
| FAT filesystem identifier | `UUID` of `/dev/nvme0n1p1` | Mount the ESP from Linux, commonly through `/etc/fstab`. |
| LUKS2 UUID | `cryptsetup luksUUID` | Select the encrypted container in `rd.luks.name=` and `rd.luks.options=`. |
| Boot-variable number | `Boot0003`, for example | Identify one NVRAM record for ordering or deletion. |

Compare the entry with the partition identity instead of comparing labels:

```bash
sudo efibootmgr -v
lsblk -no PARTUUID,UUID,FSTYPE,MOUNTPOINTS /dev/nvme0n1p1
sudo bootctl --print-loader-path
```

`BootCurrent` in the `efibootmgr` output identifies the NVRAM record used for
the current boot. The canonical entry contains the current ESP PARTUUID and
the normal systemd-boot path. If a new entry has just been created, boot
through it once before removing its predecessor; otherwise `BootCurrent`
cannot yet prove that the new record works.

After that successful boot, a record is stale only when its exact disk or GPT
partition GUID is known to be obsolete. Record its four-digit number, verify
that it is not `BootCurrent`, and remove only that NVRAM variable:

```bash
stale_bootnum=0003
boot_current=$(sudo efibootmgr | awk '/BootCurrent:/ { print $2 }')
printf 'BootCurrent=%s; candidate stale entry=%s\n' "$boot_current" "$stale_bootnum"
```

Read the values. If they are equal, stop. If they differ and the candidate has
already been proven to contain the obsolete PARTUUID, remove that exact entry
and inspect the result:

```bash
sudo efibootmgr --bootnum "$stale_bootnum" --delete-bootnum
sudo efibootmgr -v
```

This separation keeps the safety decision visible to the operator instead of
hiding it inside a multi-line shell conditional intended for a script.

The deletion changes firmware NVRAM and normally removes the number from
`BootOrder`; it does not delete the ESP, systemd-boot, or a UKI. That limited
scope makes an accidental deletion recoverable by recreating the entry or
using the signed fallback loader, but it can still make another installation
temporarily harder to start. Inspection therefore precedes deletion.

## The EFI System Partition

UEFI firmware must be able to read the executable before Linux, LUKS, LVM, and
ext4 drivers are available. The project therefore uses a FAT32 EFI System
Partition outside the LUKS2 container and mounts it directly at `/boot`.

Canonical layout:

```text
/boot/
├── EFI/
│   ├── BOOT/BOOTX64.EFI
│   ├── Linux/arch-linux.efi
│   ├── Linux/arch-linux-fallback.efi
│   └── systemd/systemd-bootx64.efi
├── loader/loader.conf
├── loader/random-seed
└── vmlinuz-linux
```

The standalone `vmlinuz-linux` is package-managed input used when rebuilding
the UKIs. No canonical boot entry executes it directly.

FAT does not store Unix ownership and mode bits. This installation mounts the
ESP with `fmask=0177,dmask=0077`, presenting regular files as root-only `0600`
and directories as root-only `0700`. Those masks reduce ordinary local
exposure after Linux starts; they are not encryption and do not prevent an
attacker with physical access from reading the FAT filesystem elsewhere.

Inspect the active mount rather than assuming `/boot` is the ESP:

```bash
findmnt /boot
findmnt -no SOURCE,FSTYPE,OPTIONS /boot
lsblk -o NAME,PATH,FSTYPE,PARTTYPE,PARTUUID,UUID,MOUNTPOINTS
```

Writing UKIs while `/boot` is accidentally unmounted would place files in the
root filesystem's ordinary `/boot` directory, leaving the real ESP unchanged.
That can produce a successful build followed by an apparently inexplicable
boot of old artifacts. Every repair checks the mount before rebuilding.

## Secure Boot verifies EFI execution, not disk confidentiality

Secure Boot is an authenticity and integrity policy enforced by UEFI when EFI
images are loaded. This project enrolls owner-controlled keys with `sbctl` and
retains Microsoft certificates for the selected hardware compatibility path.

At a high level:

- the Platform Key controls ownership of Secure Boot configuration;
- the Key Exchange Key set authorizes updates to signature databases;
- `db` contains accepted signing certificates or hashes;
- `dbx` contains revoked images or certificates.

The private owner keys stored below `/var/lib/sbctl/keys` are signing authority.
They must not be copied to the ESP, committed to Git, or stored unencrypted
beside the laptop. Their protected backup and restoration will receive a
separate trust guide.

Secure Boot in this design protects the executed systemd-boot copies and the
UKIs. It does not:

- encrypt the ESP;
- hide the boot menu or filenames;
- encrypt root, home, or swap;
- validate every file in the mounted ext4 root after boot;
- stop an attacker who has the LUKS passphrase and administrator access;
- replace firmware updates, strong account authentication, or backups.

LUKS and Secure Boot solve different problems. LUKS protects data at rest until
unlock. Secure Boot restricts which signed early-boot code and embedded boot
configuration the firmware will execute.

Verify the current policy and saved signatures with:

```bash
sudo sbctl status
sudo sbctl verify
```

An unsigned `/boot/vmlinuz-linux` report is expected in this project because it
is rebuild input, not an executable boot entry. Both systemd-boot copies and
both UKIs must be signed.

## Why systemd-boot is retained

A UEFI firmware can execute a UKI directly. systemd-boot is not technically
required between firmware and systemd-stub. The project retains it because it
provides:

- predictable Type #2 UKI discovery below `/EFI/Linux`;
- a three-second menu;
- normal and broad fallback choices;
- firmware and Boot Loader Interface integration;
- a consistent update path for the normal and UEFI fallback loader copies.

The configuration is intentionally small:

```text
default arch-linux.efi
timeout 3
console-mode max
editor no
```

`editor no` prevents interactive editing of the selected entry's kernel command
line from the menu. The canonical command line is embedded in each signed UKI.

systemd-boot discovers these as Boot Loader Specification Type #2 entries:

```text
/EFI/Linux/arch-linux.efi
/EFI/Linux/arch-linux-fallback.efi
```

No Type #1 file such as `/loader/entries/arch.conf` is needed. Adding a second
entry mechanism would create redundant paths that could diverge.

Inspect discovery and effective selection:

```bash
sudo bootctl list
sudo bootctl status
sudo cat /boot/loader/loader.conf
```

An EFI-variable default or one-shot choice can override the file configuration.
Therefore `loader.conf` alone does not always prove what the next boot will
select.

## A UKI is one EFI executable with several sections

A unified kernel image combines boot-critical components into one PE/COFF EFI
executable built around systemd-stub. Depending on its build, sections include:

- Linux kernel;
- initramfs;
- CPU microcode;
- embedded kernel command line;
- operating-system release metadata;
- kernel version metadata;
- systemd-stub and optional visual resources.

Signing the assembled UKI makes those embedded components part of the signed
EFI artifact. It avoids a boot entry that separately references an unsigned
initramfs or editable command-line file.

This project builds two images:

| UKI | Difference | What it is not |
| --- | --- | --- |
| `arch-linux.efi` | Uses `autodetect` to include the modules expected for the detected machine. | Not a minimal rescue environment. |
| `arch-linux-fallback.efi` | Skips `autodetect`, so the initramfs contains a broader module set. | Not an older kernel or package rollback. |

Both use the same installed kernel, command line, hooks, and root filesystem.
If only the fallback UKI boots, the broader early-driver set is diagnostic
evidence. Continue using it only long enough to repair the normal image.

Inspect contents and metadata without executing the image:

```bash
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
```

Both must be identified as `uki`, and their embedded command line must match
the reviewed `/etc/kernel/cmdline` source used to build them.

## How mkinitcpio and ukify produce the UKIs

The Arch `linux` package supplies `/boot/vmlinuz-linux` and a mkinitcpio preset.
The project preset defines two UKI outputs:

```text
/boot/EFI/Linux/arch-linux.efi
/boot/EFI/Linux/arch-linux-fallback.efi
```

`mkinitcpio -P` processes both presets. Mkinitcpio assembles early userspace
from the configured hooks, and ukify combines it with the kernel, microcode,
metadata, command line, and EFI stub. The sbctl integration signs the resulting
UKIs after owner keys exist.

The systemd-based hook sequence is:

```text
base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck
```

Important dependencies in that order are:

1. keyboard and virtual-console support must exist before passphrase input;
2. block drivers must expose the NVMe partition;
3. `sd-encrypt` must open LUKS before LVM metadata is accessible;
4. `lvm2` must activate `vg0` before `vg0/root` exists;
5. filesystem and fsck support must prepare ext4 before root is mounted.

The fallback preset omits `autodetect` during its build. It does not use a
different hook architecture.

Do not mix `sd-encrypt` with the BusyBox-oriented `encrypt` hook or copy kernel
parameters intended for the other initramfs path. Similar names do not make
their syntax interchangeable.

## The embedded kernel command line is the handoff contract

The project source file contains three essential parameters:

```text
rd.luks.name=<LUKS-UUID>=cryptlvm root=/dev/mapper/vg0-root rw
```

Each value has one job:

| Parameter | Effect |
| --- | --- |
| `rd.luks.name=<LUKS-UUID>=cryptlvm` | Find that LUKS2 container and open it as `/dev/mapper/cryptlvm`. |
| `root=/dev/mapper/vg0-root` | Select the root logical volume after LVM activation. |
| `rw` | Request the normal read-write root after early checking. |

The LUKS UUID must come from the target container itself:

```bash
sudo cryptsetup luksUUID /dev/nvme0n1p2
```

It is not interchangeable with:

| Identifier | Identifies |
| --- | --- |
| GPT `PARTUUID` | A partition-table entry. |
| LUKS UUID | The LUKS container metadata. |
| LVM PV/VG/LV UUID | Objects inside the decrypted device-mapper layer. |
| ext4 UUID | A filesystem inside an LV. |
| Filesystem label | A human-assigned filesystem name. |

Copying a UUID from the other ThinkPad can still produce a syntactically valid
UKI that never finds its encrypted root.

The canonical line omits `resume=` because hibernation is not configured. It
initially omits discard propagation, `quiet`, and splash options so storage and
early-boot behavior remain visible during validation. Later post-install policy
adds only the reviewed LUKS discard setting before rebuilding and re-signing
both UKIs.

Inspect the source, embedded value, and running value separately:

```bash
cat /etc/kernel/cmdline
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
cat /proc/cmdline
```

They answer three different questions: what the next rebuild will embed, what
the current file contains, and what this boot actually received.

## Entering the initramfs

After systemd-boot selects the signed UKI, systemd-stub loads its embedded
components and transfers control to the Linux kernel. The real ext4 root is not
available yet, so the kernel starts an initial userspace from the embedded
initramfs.

This early environment has its own systemd manager, units, device events,
generators, logs, and limited tools. It is temporary but fully responsible for
creating the storage path needed by the installed system.

The approximate early sequence is:

1. kernel initializes memory, drivers, and device discovery;
2. initramfs systemd starts its early targets;
3. the cryptsetup generator reads the embedded command line;
4. the NVMe partition matching the LUKS UUID is located;
5. a passphrase prompt obtains a key and cryptsetup opens `cryptlvm`;
6. LVM reads metadata through the decrypted mapping and activates `vg0`;
7. `/dev/mapper/vg0-root` becomes available;
8. ext4 is checked according to policy and mounted as the new root;
9. initramfs performs the switch-root handoff.

The LUKS prompt appearing between systemd status messages is therefore normal:
it is one dependency in the initramfs boot transaction. If no splash program is
installed, early status output can continue before and after the prompt.

## What the LUKS passphrase does

The passphrase does not directly encrypt every disk sector. LUKS metadata uses
the entered passphrase through its configured key derivation process to unlock
one of the protected keyslots, which provides access to the volume encryption
key. Device mapper then exposes decrypted block I/O through:

```text
/dev/mapper/cryptlvm
```

The underlying partition remains encrypted. The mapping exists only while it
is open in the running system.

This project keeps a strong passphrase as the canonical recovery credential.
TPM2-assisted unlocking is deferred. A later TPM design must preserve a tested
passphrase path and define which measured boot state authorizes automatic
release; convenience alone is not a sufficient policy.

Inspect the live mapping after boot:

```bash
sudo cryptsetup status cryptlvm
ls -l /dev/mapper/cryptlvm
```

Successful unlock proves access to decrypted blocks. It does not prove that LVM
metadata or ext4 is healthy.

## LVM appears only after decryption

The project uses LVM-on-LUKS. The physical volume is created on
`/dev/mapper/cryptlvm`, so its metadata is inaccessible until LUKS is open.

The layers are:

```text
/dev/nvme0n1p2
└── LUKS2 mapping: /dev/mapper/cryptlvm
    └── LVM volume group: vg0
        ├── logical volume: root
        ├── logical volume: home
        └── logical volume: swap
```

Once `lvm2` activates the group, stable device-mapper paths include:

```text
/dev/mapper/vg0-root
/dev/mapper/vg0-home
/dev/mapper/vg0-swap
```

The initramfs needs only the root LV to continue. Home and disk swap can be
activated or mounted by the normal system after switch-root.

Inspect without changing metadata:

```bash
sudo pvs
sudo vgs
sudo lvs -a -o +devices
lsblk -f
```

Do not run `pvcreate`, `vgcreate`, `lvcreate`, filesystem format, or repair
commands while diagnosing a missing LV. Those are creation or mutation tools,
not discovery commands.

## Mounting root and switching userspace

After `/dev/mapper/vg0-root` exists, the initramfs checks and mounts the ext4
filesystem as the future `/`. The system then performs a switch-root operation:
the temporary initramfs root is replaced by the installed root filesystem, and
the normal systemd manager continues the boot as PID 1.

The installed `/etc/fstab` participates after this handoff. In the canonical
system it describes:

- ext4 root;
- ext4 home;
- FAT32 ESP at `/boot` with restrictive masks;
- the disk-backed swap LV.

The embedded `root=` parameter is what lets early userspace find the first root
filesystem. An `/etc/fstab` inside that still-unmounted root cannot solve an
incorrect `root=` parameter by itself.

After switch-root, systemd reaches targets, mounts remaining filesystems,
activates swap and services, starts NetworkManager and time synchronization,
and finally presents the TTY or reviewed graphical login path.

Verify the live layers:

```bash
findmnt /
findmnt /home
findmnt /boot
swapon --show
systemctl --failed --no-pager
```

These commands confirm the running system. They do not prove that the next UKI
rebuild or next boot will use identical artifacts.

## Why logs continue around the passphrase prompt

Without a bootsplash, systemd prints unit progress while initramfs jobs start in
parallel. Storage discovery may begin, the encrypted device becomes available,
the passphrase prompt appears, and unrelated early jobs may continue emitting
status before the encrypted-root dependency completes.

That behavior is not evidence that encryption was bypassed. The decisive test
is that root cannot be mounted until `cryptlvm` is successfully opened and
`vg0/root` activated.

Plymouth can later provide a consistent graphical presentation and integrate
the LUKS prompt. It would be included in the initramfs and therefore in the
signed UKI. It changes appearance and input presentation, not the cryptographic
requirement or storage order. We will add it only after preserving:

- a visible fallback and recovery path;
- reliable passphrase entry with the US early keymap;
- signed normal and fallback UKI rebuilds;
- useful diagnostics when splash is disabled.

## Updating the boot chain

Several package updates can change boot artifacts:

- `linux` changes the kernel input;
- `amd-ucode` changes early CPU microcode;
- `mkinitcpio` or its hooks change initramfs construction;
- `systemd` can change systemd-stub, systemd-boot, ukify, and early userspace;
- `cryptsetup` or `lvm2` can change early storage tooling;
- `sbctl` integration controls signing records and post-generation signing.

The intended update flow is:

1. pacman installs updated package inputs;
2. transaction hooks run mkinitcpio presets;
3. normal and fallback UKIs are rebuilt;
4. sbctl signs the new UKIs through saved records/hooks;
5. the signed systemd-boot source is refreshed as required;
6. `systemd-boot-update.service` distributes the packaged signed loader to ESP
   locations on its update path;
7. the administrator verifies entries and signatures before rebooting.

Verification:

```bash
sudo bootctl list
sudo sbctl verify
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
```

Do not reboot after a boot-related transaction that failed to generate or sign
a required artifact. The currently running kernel does not prove the new ESP
contents are bootable.

## Normal, fallback, recovery, and rollback

These mechanisms solve different failure classes:

| Mechanism | Can help when | Cannot provide |
| --- | --- | --- |
| Normal UKI | Expected drivers and configuration work. | Broad driver recovery. |
| Fallback UKI | Normal initramfs omitted an early module. | An older kernel or older packages. |
| UEFI fallback loader path | NVRAM boot entry is missing or ignored. | A different Linux userspace. |
| Arch ISO and chroot | Installed boot artifacts or configuration need offline repair. | Automatic preservation of local secrets or data. |
| Package cache | A compatible earlier package archive is retained. | Atomic whole-system rollback. |
| Restic backup | Tracked data and configuration can be restored. | A directly bootable block snapshot by itself. |

The existence of `arch-linux-fallback.efi` should not be mistaken for an A/B
system. Both UKIs are normally regenerated during the same kernel transaction.
A bad kernel can therefore affect both.

## Diagnose by the last completed boundary

| Symptom | Last proven stage | Inspect next |
| --- | --- | --- |
| Firmware does not list Linux Boot Manager | Firmware initialized | NVRAM entry, ESP GPT type, loader files, UEFI mode. |
| Security violation before the menu | Firmware found an EFI file | Secure Boot enrollment and systemd-boot signature. |
| systemd-boot menu opens but UKI is rejected | Boot manager executed | Selected UKI signature and file integrity. |
| UKI starts but no LUKS prompt appears | Firmware, loader, UKI, kernel began | Embedded command line, initramfs hooks, block driver, fallback UKI. |
| Passphrase is rejected | LUKS device and prompt exist | Keyboard layout, correct passphrase, LUKS header and keyslots. |
| LUKS opens but root LV is missing | Decryption succeeded | `lvm2` hook, VG/LV metadata, `root=` mapper path. |
| Root LV exists but mount fails | Firmware through LVM succeeded | ext4 state, fsck result, root filesystem UUID/path. |
| Root mounts but boot enters emergency mode | Switch-root likely completed | `/etc/fstab`, required mounts, failed units, normal-system journal. |
| Login appears but desktop fails | Complete base boot succeeded | User manager, greetd/Niri, portals, graphics, user journal. |

This boundary method prevents destructive work at the wrong layer. A missing
NVRAM entry does not justify recreating LUKS; a wrong `root=` parameter does not
justify formatting ext4.

## Inspection checklist from a working boot

Use a working boot to record evidence before changing anything:

```bash
sudo efibootmgr -v
sudo bootctl status
sudo bootctl list
sudo bootctl --print-loader-path
sudo sbctl status
sudo sbctl verify
cat /proc/cmdline
sudo cryptsetup status cryptlvm
sudo pvs
sudo vgs
sudo lvs -a -o +devices
findmnt /
findmnt /home
findmnt /boot
systemctl --failed --no-pager
```

The outputs may reveal UUIDs, host information, disk layout, and boot paths.
Review them before publishing.

## Safe recovery from the Arch ISO

The recovery principle is to reconstruct the same layers without recreating
them:

1. boot the current official Arch ISO in UEFI mode;
2. identify the target disk from read-only inventory;
3. open the existing LUKS2 container as `cryptlvm`;
4. activate existing LVM volumes;
5. mount root, home, and the ESP at the same hierarchy under `/mnt`;
6. enter `arch-chroot /mnt`;
7. inspect source configuration and the mounted ESP;
8. rebuild both UKIs, sign them, and reinstall/update systemd-boot only when the
   diagnosis requires it;
9. verify every required signature and boot entry before leaving the chroot;
10. unmount and close mappings cleanly.

Opening, activating, mounting, inspecting, rebuilding, and signing are not the
same as repartitioning, formatting, `luksFormat`, `pvcreate`, or `vgcreate`.
Never substitute creation commands into a recovery sequence.

If Secure Boot prevents the ISO or repaired loader from starting, temporarily
disabling enforcement may be a diagnostic bridge. Do not clear firmware keys,
regenerate owner keys, or permanently abandon Secure Boot merely to hide an
unsigned artifact. Repair trust, verify it, and restore the intended firmware
state.

## What this architecture deliberately defers

- TPM2-assisted LUKS unlocking and PCR policy;
- hibernation and `resume=`;
- boot counting and automatic kernel rollback;
- a separate XBOOTLDR partition;
- measured boot and remote attestation;
- Plymouth splash and themed passphrase presentation;
- root-filesystem integrity systems such as dm-verity;
- alternative boot managers and direct firmware-to-UKI boot.

Each option changes a different boundary. They should not be introduced as one
large “make boot better” step.

## Sources

- [systemd-boot(7)](https://man.archlinux.org/man/systemd-boot.7)
- [systemd-stub(7)](https://man.archlinux.org/man/systemd-stub.7)
- [bootctl(1)](https://man.archlinux.org/man/bootctl.1)
- [efibootmgr(8)](https://man.archlinux.org/man/efibootmgr.8)
- [loader.conf(5)](https://man.archlinux.org/man/loader.conf.5)
- [kernel-command-line(7)](https://man.archlinux.org/man/kernel-command-line.7)
- [mkinitcpio(8)](https://man.archlinux.org/man/mkinitcpio.8)
- [mkinitcpio.conf(5)](https://man.archlinux.org/man/mkinitcpio.conf.5)
- [ukify(1)](https://man.archlinux.org/man/ukify.1)
- [systemd-cryptsetup-generator(8)](https://man.archlinux.org/man/systemd-cryptsetup-generator.8)
- [cryptsetup(8)](https://man.archlinux.org/man/cryptsetup.8)
- [lvm(8)](https://man.archlinux.org/man/lvm.8)
- [sbctl(8)](https://man.archlinux.org/man/sbctl.8)
- [Boot Loader Specification](https://uapi-group.org/specifications/specs/boot_loader_specification)
- [ArchWiki: Arch boot process](https://wiki.archlinux.org/title/Arch_boot_process)
- [ArchWiki: EFI system partition](https://wiki.archlinux.org/title/EFI_system_partition)
- [ArchWiki: Unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [ArchWiki: systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)
- [ArchWiki: dm-crypt system configuration](https://wiki.archlinux.org/title/Dm-crypt/System_configuration)
- [ArchWiki: LVM](https://wiki.archlinux.org/title/LVM)
