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

### Boot and trust

- [From UEFI firmware to the mounted root filesystem](boot-and-trust/05-uefi-to-root-boot-chain.md)
  traces firmware discovery, Secure Boot verification, systemd-boot, UKIs,
  initramfs, LUKS, LVM, root mounting, systemd handoff, updates, and recovery.
- [Secure Boot trust, signing, and recovery](boot-and-trust/07-secure-boot-trust-signing-and-recovery.md)
  explains PK, KEK, db, dbx, Setup Mode, owner and Microsoft certificates,
  signed UKIs, sbctl state, update safety, threat boundaries, and recovery.

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

### Niri ecosystem

- [Wayland, Niri, and the graphical session](niri-ecosystem/14-wayland-niri-and-session-architecture.md)
  explains the compositor model, session startup, environment propagation,
  sockets, native and X11 clients, application identity, window rules,
  outputs, IPC, layer-shell components, portals, and GNU Stow boundaries.
- [Waybar, Fuzzel, Mako, Eww, and shell evolution](niri-ecosystem/15-session-components-and-shell-evolution.md)
  separates bars, launchers, notification daemons, widget systems, lockers,
  idle coordinators, and complete shells; it records the safe path toward a
  more polished lock screen, notification center, and automatic idle suspend.

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
   suspend.
4. **Wayland and Niri:** compositor and XWayland concepts; Niri and GNU Stow;
   Mako, Waybar, Fuzzel, Eww, and component boundaries; greetd, swaylock, and
   the graphical-session lifecycle; outputs and per-host overrides.
5. **Maintenance and extensions:** Restic and restore drills; journal-led
   diagnosis and chroot recovery; update workflow; Plymouth and boot
   appearance; TPM2-bound unlock; theming and desktop polish.

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
