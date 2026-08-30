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
