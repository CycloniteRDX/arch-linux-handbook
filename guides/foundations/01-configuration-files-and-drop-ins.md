# Configuration files, drop-ins, and precedence

## Purpose and scope

Linux configuration is not governed by one universal rule. `/etc` usually
contains administrator policy, `.d` directories usually contain small
fragments, and later filenames often win, but each program defines what it
reads and in which order.

This guide provides a safe method for answering four questions before changing
a system:

1. Which program reads this file?
2. Does the file belong to a package, the administrator, or the user?
3. How are multiple sources combined?
4. How can the effective result be inspected and reverted?

It explains configuration layout, not every option supported by the programs
used in this project.

## The three main ownership layers

| Location | Typical owner | Intended content |
| --- | --- | --- |
| `/usr` | Distribution packages | Executables, libraries, unit files, and vendor defaults. |
| `/etc` | System administrator | Machine-wide policy and overrides. |
| `~/.config` | Individual user | Per-user application and desktop configuration. |

These are conventions, not proof. A program may use another path, and a file
under `/etc` may still have been installed by a package. Ask pacman instead of
guessing:

```bash
pacman -Qo /etc/tlp.conf
pacman -Qo /usr/lib/systemd/system/fstrim.service
```

`pacman -Qo` reports which installed package owns a path. A locally created
file normally has no package owner. `pacman -Qkk package` performs a detailed
check of the files recorded for an installed package, including metadata where
the package provides it.

Do not edit files under `/usr` to express local policy. Package upgrades may
replace them. Prefer a documented file under `/etc`, or a user file under
`~/.config` when the setting is personal.

## Main files and drop-ins

A main configuration file collects settings in one place. A *drop-in* is a
smaller fragment loaded in addition to a main file, commonly from a directory
whose name ends in `.d`.

Drop-ins are useful because they:

- isolate one local decision;
- show which settings the project owns;
- avoid copying a long vendor example that can become stale;
- make review and rollback smaller;
- allow packages and administrators to provide separate layers.

They are not automatically better in every case. Use a main file when the
program only supports one, when the complete file is the natural unit of
configuration, or when splitting it would obscure rather than clarify the
result. PAM files such as `/etc/pam.d/login`, `/etc/kernel/cmdline`,
`/etc/mkinitcpio.conf`, `/etc/systemd/zram-generator.conf`, and greetd's TOML
configuration are examples in this project that must be treated according to
their own manuals.

The filename prefix is not magic. `10-feature.conf` and `70-policy.conf` are
human-readable ordering choices. They matter only if the program reads files
lexicographically, and they do not override a different precedence rule.

## systemd manager configuration

Systemd manager configuration, including `logind.conf`, supports main files
and `.conf.d` drop-ins. For this family:

- the main file is read before its drop-ins;
- drop-in filenames are combined and sorted lexicographically across the
  documented directories;
- for a scalar option, the last applicable assignment normally wins;
- list-valued options may accumulate and sometimes require an empty assignment
  before rebuilding the list;
- an identically named file under `/etc` has precedence over the corresponding
  file under `/run` or `/usr`.

This project therefore keeps its lid and suspend policy in:

```text
/etc/systemd/logind.conf.d/70-thinkpad-suspend.conf
```

It is a local administrator fragment, not a file supplied by systemd. The
number places it in the range conventionally used for local policy, but the
option semantics and documented load order are what determine the result.

Inspect the combined configuration before and after changing it:

```bash
systemd-analyze cat-config systemd/logind.conf
```

After editing logind policy, validate the file and follow the activation or
reboot procedure documented by the relevant post-install chapter. Do not
restart `systemd-logind` casually from the graphical session: doing so can
terminate sessions.

## Systemd unit overrides are related, but different

A service unit such as `example.service` can be extended by fragments under:

```text
/etc/systemd/system/example.service.d/*.conf
```

Create a local unit override with:

```bash
sudo systemctl edit example.service
```

Inspect the vendor unit and every applied drop-in with:

```bash
systemctl cat example.service
```

Unit directives have type-specific rules. Some replace earlier values; others
form lists. For example, replacing an `ExecStart=` supplied by a service often
requires an empty `ExecStart=` assignment followed by the replacement. Always
read the manual for the unit section and directive being changed.

After creating or removing unit files or their drop-ins, reload the unit model:

```bash
sudo systemctl daemon-reload
```

Reloading definitions does not by itself restart a running service.

## TLP uses a different order

TLP demonstrates why `.d` must not be interpreted using systemd rules. Its
current order is:

1. intrinsic defaults;
2. `/etc/tlp.d/*.conf`, in lexical order;
3. `/etc/tlp.conf`, read last.

The last occurrence of the same parameter wins. Consequently, an active value
in `/etc/tlp.conf` overrides the same parameter in every TLP drop-in, even one
named `99-local.conf`.

This project creates:

```text
/etc/tlp.d/10-thinkpad-battery.conf
```

The file isolates the chosen battery thresholds. It works as intended only
while `/etc/tlp.conf` does not activate conflicting values. Inspect TLP's
actual sources and differences from defaults with:

```bash
sudo tlp-stat -c
sudo tlp-stat --cdiff
```

Apply reviewed TLP changes with the activation method documented upstream and
in the hardware-and-power chapter; for example, `sudo tlp start` applies the
current configuration without requiring a reboot.

## User configuration and GNU Stow

The same ownership principle applies without root. Niri, Kitty, Mako, Waybar,
Fuzzel, swaylock, and `mimeapps.list` use files below `~/.config` in this
project. GNU Stow deploys reviewed repository files there as symbolic links.

Inspect where a deployed path leads:

```bash
readlink -f ~/.config/niri/config.kdl
stat ~/.config/niri/config.kdl
```

An application's graphical preferences may edit the tracked file through that
link. Review the repository after using a **Make default** or settings action:

```bash
git status --short
git diff --check
git diff
```

Do not assume every application follows the XDG Base Directory specification
or supports fragments. Confirm its documented paths before adding another Stow
package.

## Package updates and `.pacnew`

Pacman treats selected configuration files as backup files. If both the
installed file and the package's new version changed, pacman preserves the
active file and writes the new package version with a `.pacnew` suffix. It does
not merge the files automatically.

Find pending files with:

```bash
sudo pacdiff --output
```

Review each pair and merge deliberately. Never replace all active files with
their `.pacnew` versions as a batch. A new package version may add an important
default, while the existing file contains local policy that still matters.

Drop-ins reduce conflicts with main files, but they do not eliminate review:
new vendor defaults or renamed options can change the meaning of a local
fragment.

## A safe change workflow

Use this sequence for any unfamiliar configuration system:

1. Read the program's current manual and identify every input file.
2. Use `pacman -Qo` to distinguish package-owned files from local ones.
3. Inspect the effective configuration with the program's own tool, when one
   exists.
4. Back up a risky main file before editing it; do not place backups where the
   program will parse them as additional configuration.
5. Make the smallest coherent change in the documented administrator or user
   layer.
6. Run syntax or semantic validation before reload, restart, or reboot.
7. Inspect the effective result again and test the affected behavior.
8. Record the change in the appropriate repository without credentials or
   machine secrets.

## Rollback

For a local drop-in, rollback usually means moving the exact fragment out of
the parsed directory, reloading the relevant component, and verifying the
combined result. Do not delete an unknown file merely because it is under
`/etc`; identify its owner and purpose first.

For a Stow package, preview removal and then remove only its links:

```bash
stow --simulate --delete --verbose --no-folding --target="$HOME" niri
stow --delete --verbose --no-folding --target="$HOME" niri
```

The repository file remains available for redeployment. A real file that was
not created by Stow must be handled separately and should be backed up before
replacement.

## Project examples at a glance

| Path | Kind | Important rule |
| --- | --- | --- |
| `/etc/tlp.d/10-thinkpad-battery.conf` | TLP drop-in | `/etc/tlp.conf` is read after all TLP drop-ins. |
| `/etc/systemd/logind.conf.d/70-thinkpad-suspend.conf` | systemd manager drop-in | Drop-ins follow the main file; inspect with `systemd-analyze cat-config`. |
| `/etc/systemd/system/*.service.d/*.conf` | systemd unit drop-in | Directive-specific merge rules apply; inspect with `systemctl cat`. |
| `/etc/systemd/zram-generator.conf` | Main configuration | The project uses a complete program-specific file, not a `.d` fragment. |
| `/etc/pam.d/login` | PAM service stack | Order is security-sensitive; back up and test a second login before rebooting. |
| `~/.config/niri/config.kdl` | User configuration via Stow | The live path is a symlink to a reviewed dotfiles file. |

## Sources

- [systemd-system.conf(5): configuration directories and precedence](https://man.archlinux.org/man/systemd-system.conf.5)
- [systemd.unit(5): unit load paths and drop-ins](https://man.archlinux.org/man/systemd.unit.5)
- [systemd-analyze(1): `cat-config`](https://man.archlinux.org/man/systemd-analyze.1)
- [TLP configuration introduction](https://linrunner.de/tlp/settings/introduction.html)
- [pacman(8): queries and handling configuration files](https://man.archlinux.org/man/pacman.8)
- [pacdiff(8)](https://man.archlinux.org/man/pacdiff.8)
- [ArchWiki: XDG Base Directory](https://wiki.archlinux.org/title/XDG_Base_Directory)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)
