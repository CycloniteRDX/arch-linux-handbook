# Handbook index

This index is both a reading map and a writing backlog. A directory is created
only when its first reviewed article is ready.

## Published guides

### Foundations

- [Configuration files, drop-ins, and precedence](foundations/01-configuration-files-and-drop-ins.md)
  explains `/usr`, `/etc`, user configuration, program-specific precedence,
  package updates, inspection, and safe rollback.
- [Users, permissions, sudo, and PAM](foundations/02-users-permissions-sudo-and-pam.md)
  separates identity, file access, delegated privilege, authentication stacks,
  graphical authorization, and safe recovery.
- [Systemd units, activation, and the journal](foundations/03-systemd-units-activation-and-journal.md)
  explains unit types, runtime and enablement states, dependencies, services,
  sockets, timers, targets, user managers, generators, logs, and recovery.
- [Package lifecycle, upgrades, and the AUR](foundations/04-package-lifecycle-upgrades-and-aur.md)
  explains pacman transactions, package state and queries, complete upgrades,
  configuration merges, cache, mirrors, foreign packages, AUR review, and
  recovery.
- [Shell, terminal, paths, redirection, editors, and
  documentation](foundations/21-shell-terminal-paths-and-documentation.md)
  distinguishes terminal and shell layers and explains Bash parsing, paths,
  quoting, environments, standard streams, exit statuses, jobs, safe file
  operations, Micro, startup files, PowerShell boundaries, and how to read
  local and online documentation.

### Git and GitHub

- [Git, GitHub, SSH authentication, and repository
  workflow](git-and-github/22-git-github-ssh-and-repository-workflow.md)
  distinguishes local version control, remote hosting, author identity, SSH
  authentication, and authorization; it explains repository state, deliberate
  staging, Conventional Commits, branches, synchronization, conflicts,
  cross-platform policy, and safe recovery.

### Boot and trust

- [From UEFI firmware to the mounted root filesystem](boot-and-trust/05-uefi-to-root-boot-chain.md)
  traces firmware discovery, Secure Boot verification, systemd-boot, UKIs,
  initramfs, LUKS, LVM, root mounting, systemd handoff, updates, and recovery.
- [Secure Boot trust, signing, and recovery](boot-and-trust/07-secure-boot-trust-signing-and-recovery.md)
  explains PK, KEK, db, dbx, Setup Mode, owner and Microsoft certificates,
  signed UKIs, sbctl state, update safety, threat boundaries, and recovery.
- [Plymouth, early-boot presentation, and recovery](boot-and-trust/24-plymouth-early-boot-presentation-and-recovery.md)
  explains firmware-to-greeter visual handoffs, Plymouth architecture,
  graphical LUKS requests, KMS and SimpleDRM, quiet versus splash, distinct
  graphical and textual UKIs, themes, signing, testing, diagnosis, and
  recovery.
- [TPM2-bound LUKS unlock, measured-boot policy, and recovery](boot-and-trust/25-tpm2-bound-luks-unlock-measured-boot-and-recovery.md)
  separates Secure Boot, measured boot, TPM sealing, LUKS, and login; it
  explains PCRs, signed PCR 11 policy, PCR 7, PIN and fallback credentials,
  UKI policy signatures, updates, enrollment staging, diagnosis, rollback,
  and recovery.

### Storage

- [Storage layers, encryption, memory, and discard](storage/06-storage-layers-encryption-memory-and-trim.md)
  explains NVMe, GPT, stable identifiers, LUKS2, dm-crypt, LVM, ext4,
  persistent mounts, disk swap, zram, TRIM, SSD spare space, and safe recovery.

### Networking

- [NetworkManager, addressing, routes, and DNS](networking/08-networkmanager-addressing-routes-and-dns.md)
  explains devices and profiles, Wi-Fi, DHCP, IPv4 and IPv6, routes, DNS,
  `/etc/resolv.conf`, local resolvers, privacy boundaries, and layered diagnosis.
- [Firewalld, nftables, zones, and host exposure](networking/09-firewalld-nftables-zones-and-host-exposure.md)
  explains stateful filtering, zone assignment, services and ports, runtime and
  permanent state, forwarding, NAT, NetworkManager integration, and diagnosis.

### Workstation

- [Polkit authorization and XDG Desktop Portals](workstation/10-polkit-and-xdg-desktop-portals.md)
  separates privileged action authorization, authentication agents, portal
  brokers and backends, session activation, file grants, screenshots,
  screencasting, notifications, secrets, verification, and recovery.
- [PipeWire, WirePlumber, and audio routing](workstation/11-pipewire-wireplumber-and-audio-routing.md)
  explains ALSA and Bluetooth boundaries, the PipeWire graph, WirePlumber
  policy, profiles, routes, defaults, compatibility APIs, state, latency,
  verification, and recovery.
- [Bluetooth, removable media, and Secret Service](workstation/12-bluetooth-removable-media-and-secret-service.md)
  explains BlueZ devices and trust, UDisks authorization, udiskie automount,
  GIO/GVfs and MTP, GNOME Keyring, Secret Service, PAM unlock paths, safe
  removal, verification, and recovery.
- [TLP, logind, idle handling, and suspend](workstation/13-tlp-logind-idle-and-suspend.md)
  explains power-policy ownership, TLP profiles, ThinkPad charge thresholds,
  platform profiles, lid handling, sleep states, inhibitors, pre-suspend
  locking, resume verification, and layered diagnosis.
- [XDG directories, desktop entries, MIME associations, fonts, and locales](workstation/18-xdg-directories-desktop-entries-mime-and-fonts.md)
  explains base and user directories, portable versus generated state,
  locale and keyboard boundaries, application metadata, MIME detection and
  defaults, Fontconfig selection, integration diagnosis, and safe recovery.
- [Printing, scanning, and peripheral integration](workstation/23-printing-scanning-and-peripheral-integration.md)
  explains CUPS queues and jobs, driverless IPP, local versus shared roles,
  discovery and firewalld boundaries, polkit administration, IPP-over-USB,
  SANE and AirScan, udev identity, device access, power policy, diagnosis, and
  recovery.

### Niri ecosystem

- [Wayland, Niri, and the graphical session](niri-ecosystem/14-wayland-niri-and-session-architecture.md)
  explains the compositor model, session startup, environment propagation,
  sockets, native and X11 clients, application identity, window rules,
  outputs, IPC, layer-shell components, portals, and GNU Stow boundaries.
- [Waybar, Fuzzel, Mako, Eww, and shell evolution](niri-ecosystem/15-session-components-and-shell-evolution.md)
  separates bars, launchers, notification daemons, widget systems, lockers,
  idle coordinators, and complete shells; it records the safe path toward a
  more polished lock screen, notification center, and automatic idle suspend.
- [greetd, tuigreet, PAM, and graphical login recovery](niri-ecosystem/16-greetd-tuigreet-pam-and-login-recovery.md)
  traces the login boundary from the system service and terminal greeter
  through PAM, logind, `niri-session`, the user manager, logout, verification,
  and TTY or installation-media recovery.
- [Niri outputs, scaling, external displays, and host overrides](niri-ecosystem/17-niri-outputs-scaling-and-host-overrides.md)
  explains connectors and display identity, modes and refresh rates, physical
  versus logical pixels, fractional scaling, coordinates, hot-plug, lid
  behavior, workspace migration, and a portable per-ThinkPad override design.
- [Desktop polish, modular-shell architecture, and automatic suspend](niri-ecosystem/26-desktop-polish-modular-shell-and-automatic-suspend.md)
  selects a personal modular Niri desktop, separates themes, icons, cursors,
  wallpapers, notifications, widgets, locking, greeters, and idle ownership,
  and defines the reversible path toward SwayNC, Eww, improved locking, and
  battery-aware automatic suspend.

### Maintenance and recovery

- [Restic backups, retention, restore drills, and recovery media](maintenance-and-recovery/19-restic-backups-retention-and-restore-drills.md)
  explains backup boundaries, repository structure and encryption, source
  selection, integrity checks, safe restores, candidate retention, secondary
  copies, future scheduling, recovery bundles, installation media, and failure
  recovery.
- [Journal-led diagnosis, update incidents, and chroot recovery](maintenance-and-recovery/20-journal-update-incidents-and-chroot-recovery.md)
  connects bounded journal queries, package transactions, boot stages, TTY and
  ISO escalation, mounted-system inspection, chroot repair, offline ext4
  diagnosis, incident records, and the recurring maintenance cadence.

## Current roadmap

Guides 01 through 20 now form the first essential edition: the handbook
explains every major subsystem needed to operate, diagnose, maintain, and
recover the installed workstation. Guides 21 through 26 are accepted
extensions, but the system does not depend on them to be understandable or
recoverable.

| Guide | Status | Planned subject |
| --- | --- | --- |
| 16 | Published | greetd, tuigreet, PAM, session creation, logout, and graphical-login recovery. |
| 17 | Published | Niri outputs, scaling, lid/external-display behavior, and per-host overrides for the two ThinkPads. |
| 18 | Published | XDG directories, desktop entries, MIME associations, fonts, locales, and application integration. |
| 19 | Published | Restic repositories, backup boundaries, retention, restore drills, and recovery media. |
| 20 | Published | Journal-led diagnosis, update incidents, chroot and boot recovery, and the recurring maintenance workflow. |
| 21 | Published | Shell and terminal fundamentals, redirection, editors, paths, and the documentation workflow. |
| 22 | Published | Git and GitHub, SSH authentication, repository setup, daily workflow, Conventional Commits, and recovery. |
| 23 | Published | Printing and peripheral integration, discovery, authorization, drivers, and diagnosis. |
| 24 | Published | Plymouth, early-boot presentation, encrypted-root prompts, UKI integration, and recovery. |
| 25 | Published | TPM2-bound LUKS unlock, measured-boot policy, fallback credentials, and recovery. |
| 26 | Published | Themes, icons, cursors, wallpapers, improved greeter/locker/notifications, automatic suspend, and the modular-shell decision. |

The first essential edition and all six accepted extensions are complete.
New subjects may still be added when they expose a missing conceptual or
recovery boundary.

| Guide family | Planned subjects |
| --- | --- |
| Foundations | Shell basics, paths, permissions, redirection, editors, logs, and documentation workflow. |
| Git and GitHub | Installation, identity and privacy, SSH authentication, GitHub CLI, repository setup, daily workflow, Conventional Commits, and recovery. |
| Package management | Pacman concepts, safe upgrades, package queries, cache management, mirrors, Reflector, AUR boundaries, and troubleshooting. |
| systemd | Units, services, sockets, timers, targets, logs, boot analysis, and safe overrides. |
| Storage | GPT, identifiers, LUKS2, LVM, ext4, Btrfs, fstab, swap, zram, hibernation, TRIM, discard, and SSD overprovisioning. |
| Boot and trust | UEFI, EFI System Partitions, initramfs, UKI, systemd-stub, systemd-boot, Secure Boot, sbctl, TPM concepts, and EFI variables. |
| Networking | NetworkManager, DNS, Wi-Fi, SSH client and server roles, firewalls, and troubleshooting. |
| Workstation | Graphics, audio, Bluetooth, printing, XDG directories, fonts, portals, polkit, secrets, and removable media. |
| Niri ecosystem | Wayland fundamentals, Niri concepts, launchers, bars, notifications, locking, idle handling, portals, screenshots, and theming. |
| Maintenance and recovery | Backups, package incidents, chroot recovery, boot recovery, storage checks, logs, and rollback strategies. |

## Recommended writing order

The order below follows conceptual dependencies, not the post-install chapter
numbers:

1. **Foundations:** configuration sources and precedence; ownership,
   permissions, sudo, and PAM; systemd units, activation, and the journal;
   package lifecycle, `.pacnew`, mirrors, and AUR boundaries.
2. **System architecture:** the UEFI-to-root boot chain; LUKS, LVM, ext4,
   swap, zram, and TRIM; Secure Boot keys, signing, threat model, and recovery.
3. **Workstation integration:** NetworkManager and DNS; firewalld, nftables,
   and zones; polkit versus XDG Desktop Portals; PipeWire and WirePlumber;
   Bluetooth, UDisks, GVfs, Secret Service, and PAM; TLP, logind, idle, and
   suspend; XDG directories, desktop entries, MIME defaults, fonts, and
   locales.
4. **Wayland and Niri:** compositor and XWayland concepts; Niri and GNU Stow;
   Mako, Waybar, Fuzzel, Eww, and component boundaries; greetd, swaylock, and
   the graphical-session lifecycle; outputs and per-host overrides.
5. **Operator tooling:** shell and terminal fundamentals; Git's repository
   model; GitHub and SSH authentication; deliberate publication and recovery.
6. **Maintenance and extensions:** Restic and restore drills; journal-led
   diagnosis and chroot recovery; update workflow; printing and peripherals;
   Plymouth and boot appearance; TPM2-bound unlock; theming and desktop polish.

This backlog may grow when a runbook or post-install instruction relies on a
concept that deserves an explanation of its own.

## Article shape

A detailed guide should normally contain:

1. Purpose and scope.
2. Concepts and terminology.
3. Recommended approach for this project.
4. Alternatives and trade-offs.
5. Practical examples.
6. Verification.
7. Troubleshooting and recovery.
8. Sources.

Not every short article needs every heading. Structure should serve the topic,
not create filler.
