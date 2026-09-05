# Daily and occasional command cheatsheet

## Purpose and scope

This is the short operational reference for the Arch Linux workstation built
by the companion runbook, post-install, and Niri dotfiles repositories. It is
for finding a known command quickly after the underlying procedure has already
been understood and validated.

It covers:

- package queries, upgrades, installation, removal, and cache maintenance;
- files, disk space, Trash, archives, and removable storage;
- Wi-Fi, Bluetooth, audio, brightness, battery profiles, and suspend;
- processes, systemd services, logs, firewall state, and boot trust;
- firmware checks, Restic backups, Git, dotfiles, Niri, and Qt 6 appearance;
- a practical recurring-maintenance cadence.

It is not a substitute for the detailed guides. When a command gives an
unexpected result, stop and follow the linked diagnostic guide instead of
trying increasingly forceful variants.

## Conventions and safety rules

- Run commands as the regular user unless `sudo` is shown.
- Replace uppercase placeholders such as `PACKAGE`, `PROFILE`, `DEVICE_MAC`,
  and `/dev/sdXN`; do not type them literally.
- Quote Wi-Fi names, connection profiles, and paths that can contain spaces.
- Read every pacman transaction and removal target before confirming it.
- A command with no output can be a valid result. Check its manual and exit
  status before treating silence as failure.
- Preview and inspect before deleting, unmounting, disabling, or overwriting.
- Never paste public output containing Wi-Fi secrets, private paths, tokens,
  key material, or a full unreviewed system inventory.
- Do not use hibernation commands: this workstation configures suspend only.

Open local help when a placeholder or option is unclear:

```bash
man COMMAND
COMMAND --help
man -k SEARCH_TERM
info COMMAND
type -a COMMAND
command -v COMMAND
```

## Everyday quick card

### System and update overview

```bash
checkupdates
systemctl --failed --no-pager
systemctl --user --failed --no-pager
tlpctl get
nmcli -f NAME,TYPE,DEVICE connection show --active
wpctl get-volume @DEFAULT_AUDIO_SINK@
df -hT
```

`checkupdates` returning no text with exit status `2` means that no official
repository updates are available.

### Common desktop actions

```bash
swaylock -f
systemctl suspend
tlpctl balanced
tlpctl power-saver
tlpctl performance
bluetoothctl power on
bluetoothctl power off
nmcli radio wifi on
nmcli radio wifi off
```

Restore TLP's automatic profile selection for the current power source after a
manual profile test:

```bash
sudo tlp start
tlpctl get
```

### Audio and brightness

```bash
wpctl set-volume -l 1.0 @DEFAULT_AUDIO_SINK@ 5%+
wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-
wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle
wpctl set-mute @DEFAULT_AUDIO_SOURCE@ toggle
brightnessctl set +5%
brightnessctl set 5%-
```

### Trash

```bash
gio trash --list
gio trash ./PATH
gio trash --empty
```

`gio trash ./PATH` is the recoverable command-line equivalent of moving an
item to Trash. `gio trash --empty` permanently deletes the current user's
trashed items and only frees their filesystem space at that point. Always run
`gio trash --list` first.

## Pacman and official packages

For the complete policy, read [Package lifecycle, upgrades, and the
AUR](../foundations/04-package-lifecycle-upgrades-and-aur.md).

### Search and inspect

```bash
pacman -Ss 'SEARCH_TERM'
pacman -Qs 'SEARCH_TERM'
pacman -Si PACKAGE
pacman -Qi PACKAGE
pacman -Ql PACKAGE
pacman -Qo /absolute/path/to/file
```

| Command | Meaning |
| --- | --- |
| `pacman -Ss` | Search enabled repository databases. |
| `pacman -Qs` | Search installed packages. |
| `pacman -Si` | Show repository information. |
| `pacman -Qi` | Show installed-package information. |
| `pacman -Ql` | List files owned by an installed package. |
| `pacman -Qo` | Identify which installed package owns one path. |

Inspect explicit packages, dependency trees, or exceptional package state:

```bash
pacman -Qe
pactree PACKAGE
pactree -r PACKAGE
pacman -Qdt
pacman -Qm
```

- `pacman -Qdt` lists orphan candidates; it does not authorize their removal.
- `pacman -Qm` lists foreign packages, including manually installed AUR
  packages and packages removed from the enabled repositories.
- `pactree -r PACKAGE` explains which installed packages depend on the target.

### Install or upgrade

Read current [Arch Linux news](https://archlinux.org/news/) first. Perform a
complete upgrade, optionally adding reviewed packages to the same transaction:

```bash
sudo pacman -Syu
sudo pacman -Syu PACKAGE_ONE PACKAGE_TWO
```

Afterward, check configuration merges and basic system state:

```bash
sudo pacdiff --output
systemctl --failed --no-pager
systemctl --user --failed --no-pager
journalctl -b -p err..alert --no-pager
```

If the kernel, microcode, mkinitcpio, systemd, systemd-boot, sbctl, or another
boot component changed, verify before rebooting:

```bash
sudo bootctl --esp-path=/boot list
sudo sbctl verify
```

Then reboot at a time when recovery is possible:

```bash
systemctl reboot
```

After login:

```bash
uname -r
systemctl --failed --no-pager
journalctl -b -p err..alert --no-pager
sudo sbctl verify
```

### Remove a package

Inspect the package and reverse dependency tree first:

```bash
pacman -Qi PACKAGE
pactree -r PACKAGE
```

Remove the package and dependencies that are no longer required by another
installed package:

```bash
sudo pacman -Rs PACKAGE
```

Read the complete removal list. Add `-n` only when intentionally discarding
package backup files instead of retaining `.pacsave` recovery copies. Never use
`-Rdd` to force through an unexplained dependency problem.

Recheck candidates after removal:

```bash
pacman -Qdt
sudo pacdiff --output
```

Do not pipe `pacman -Qdtq` directly into a removal command. Review every
candidate and its reverse dependencies individually.

### Package cache and transaction history

The enabled `paccache.timer` already retains the three latest cached versions.
Inspect the timer and preview its normal cleanup:

```bash
systemctl status paccache.timer --no-pager
systemctl list-timers --all paccache.timer --no-pager
sudo paccache --dryrun
```

Run the same three-version cleanup manually only when needed:

```bash
sudo paccache -r
```

Inspect recent package transactions:

```bash
tail -n 100 /var/log/pacman.log
less /var/log/pacman.log
```

Do not use `pacman -Scc` for routine cleanup; it removes the cached downgrade
path. Do not use `pacman -Sy` alone, routine `-Syyu`, `--noconfirm`,
`--overwrite`, or `--nodeps` to hide an unexplained problem.

### AUR boundary

After optional post-install chapter 16, Paru is the selected convenience
client. It does not turn the AUR into a trusted binary repository. Inventory
foreign packages and pending AUR recipe versions with:

```bash
pacman -Qm
pacman -Qmq
paru -Qua
paru -P --stats
```

Search official repositories first. Inspect an exact AUR candidate before
installation:

```bash
pacman -Ss 'SEARCH_TERM'
paru --aur -Si PACKAGE
paru -Gc PACKAGE
paru -Gp PACKAGE | bat --language=bash --paging=always
```

For a previously understood package, retain recipe review and an explicit AUR
target:

```bash
paru --aur -S --review --removemake=ask PACKAGE
```

Maintain official and AUR packages in separate visible phases:

```bash
sudo pacman -Syu
sudo pacdiff --output
paru -Sua --review --upgrademenu --removemake=ask
```

For each foreign package, identify its source, review the current `PKGBUILD`
and changes since the installed build, then use the full documented build
procedure. Never run `makepkg` as root, enable `SkipReview`, or run
`sudo paru`. Never assume `sudo pacman -Syu` updates foreign packages. Use
the complete chapter 16 procedure for first installation, PGP failures,
rebuilds, cache decisions, and recovery.

Keep `~/Builds/aur/paru` as the reviewed manual recovery clone. It is not a
duplicate installation and should not be removed during routine cleanup. If a
validated `makepkg -Csri` build left temporary work behind, inspect and move
only `src/` and `pkg/` to Trash:

```bash
cd ~/Builds/aur/paru
du -sh src pkg 2>/dev/null
gio trash ./src ./pkg
```

Future manual builds use `makepkg -Ccsri`: capital `-C` cleans before the build
and lowercase `-c` cleans after a successful build.

## Files, paths, searching, and disk space

### Navigate and inspect

```bash
pwd
ls -lah
tree -a -L 2
bat FILE
readlink -f PATH
file PATH
```

Search file contents, names, and JSON:

```bash
rg -n 'PATTERN' PATH
fd 'NAME' PATH
find PATH -maxdepth 2 -type f
jq . FILE.json
```

Use `rg --hidden` only when hidden files are intentionally in scope; it can
include configuration and repository metadata that an ordinary search skips.

### Create, copy, move, and link

```bash
mkdir -p PATH
cp --interactive --archive SOURCE DESTINATION
mv --interactive SOURCE DESTINATION
ln -s SOURCE LINK_NAME
```

For a large directory copy, preview with rsync and repeat without `--dry-run`
only after checking the source, destination, and trailing slashes:

```bash
rsync --archive --human-readable --info=progress2 --dry-run SOURCE/ DESTINATION/
rsync --archive --human-readable --info=progress2 SOURCE/ DESTINATION/
```

`rsync` is not a backup by itself: a mistaken source, destination, or deletion
option can reproduce the mistake.

### Trash, permanent deletion, and restore

Move one item to Trash and inspect all current Trash entries:

```bash
gio trash ./PATH
gio trash --list
```

The list prints `trash://` URIs and original paths. Restore a selected item by
copying its exact URI from that output:

```bash
gio trash --restore 'trash:///EXACT_ITEM_URI'
```

Empty Trash only after inspecting it:

```bash
gio trash --list
gio trash --empty
```

Not every filesystem supports Trash. When permanent deletion is genuinely
required, use the narrowest exact target and avoid recursive force options:

```bash
rm --interactive FILE
rmdir EMPTY_DIRECTORY
```

### Find space usage

```bash
df -hT
du -sh PATH
du -h --max-depth=1 PATH | sort -h
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS,MODEL,TRAN
findmnt --real
```

- `df` reports filesystem capacity.
- `du` reports space reachable below a directory.
- `lsblk` reports block-device layers; it does not prove that a filesystem is
  mounted.
- `findmnt` is the authoritative mount view.

If a filesystem is busy, inspect its users instead of forcing an unmount:

```bash
sudo lsof +f -- MOUNT_POINT
```

### Archives

```bash
tar -caf archive.tar.zst DIRECTORY
tar -tf archive.tar.zst
tar -xf archive.tar.zst
zip -r archive.zip DIRECTORY
unzip -l archive.zip
unzip archive.zip
7z l archive.7z
7z x archive.7z
```

List unfamiliar archives before extracting them. Extract into a new empty
directory when their internal paths are unknown.

## Removable media

The graphical session normally starts exactly one udiskie process and Nautilus
shows mounted media:

```bash
pgrep -a -x udiskie
udisksctl status
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS,MODEL,TRAN
```

Mount or unmount one exact partition manually:

```bash
udisksctl mount -b /dev/sdXN
udisksctl unmount -b /dev/sdXN
```

Unmount all media managed for the user when that is genuinely intended:

```bash
udiskie-umount -a
```

After every partition on a removable drive is unmounted, power off the exact
whole USB disk before unplugging it:

```bash
udisksctl power-off -b /dev/sdX
```

`/dev/sdXN` is a partition; `/dev/sdX` is the whole disk. Recheck both with
`lsblk` immediately before use. Encrypted backup media also needs its mapping
locked; use the complete closing procedure in the post-install backup chapter
rather than guessing its current mapping name.

## Wi-Fi and NetworkManager

For layering and diagnostics, read [NetworkManager, addressing, routes, and
DNS](../networking/08-networkmanager-addressing-routes-and-dns.md).

### Inspect state

```bash
nmcli general status
nmcli radio
nmcli device status
nmcli -f NAME,TYPE,DEVICE connection show --active
nmcli connection show
nmcli device wifi list
```

The last two commands can reveal saved and nearby network names. Review before
sharing their output publicly.

### Connect and disconnect

Connect to a new protected network without putting the password in shell
history:

```bash
nmcli --ask device wifi connect "SSID"
```

Activate or deactivate a saved profile by its exact connection name:

```bash
nmcli connection up id "PROFILE"
nmcli connection down id "PROFILE"
```

Control only the Wi-Fi radio:

```bash
nmcli radio wifi off
nmcli radio wifi on
```

Delete a saved profile only when its configuration and stored credentials are
intentionally no longer needed:

```bash
nmcli connection delete id "PROFILE"
```

### Diagnose connectivity

```bash
nmcli networking connectivity check
nmcli device show
ip -brief address
ip route
cat /etc/resolv.conf
journalctl -b -u NetworkManager.service --no-pager
```

Interpret the layers in order: radio, device, active profile, address, route,
DNS, then application. This project does not enable `systemd-resolved`, so
`resolvectl` is not the canonical DNS check.

Inspect the firewalld zone recorded by an active profile:

```bash
nmcli -f NAME,TYPE,DEVICE connection show --active
nmcli -g connection.zone connection show "ACTIVE_PROFILE"
```

An empty zone is valid in this project: firewalld then applies its default
zone. Never publish output produced with `--show-secrets`.

## Bluetooth

For the full device, trust, and audio model, read [Bluetooth, removable media,
and Secret Service](../workstation/12-bluetooth-removable-media-and-secret-service.md).

### Inspect and control the adapter

```bash
systemctl is-active bluetooth.service
bluetoothctl show
bluetoothctl devices
rfkill list bluetooth
```

Keep the BlueZ service enabled and normally control only adapter power:

```bash
bluetoothctl power on
bluetoothctl power off
```

If the adapter has only a software block:

```bash
sudo rfkill unblock bluetooth
rfkill list bluetooth
```

Do not use `rfkill unblock all`; it can alter deliberate Wi-Fi or WWAN state.

### Pair a new device

Start the interactive client:

```bash
bluetoothctl
```

Then enter:

```text
power on
agent on
default-agent
scan on
devices
pair DEVICE_MAC
trust DEVICE_MAC
connect DEVICE_MAC
scan off
quit
```

Confirm pairing codes on both devices and never trust an unidentified address.

### Reconnect, inspect, or forget a known device

```bash
bluetoothctl info DEVICE_MAC
bluetoothctl connect DEVICE_MAC
bluetoothctl disconnect DEVICE_MAC
bluetoothctl remove DEVICE_MAC
```

`remove` forgets the pairing and trust record. Use `disconnect` when the device
should remain paired.

For Bluetooth audio, confirm that PipeWire sees the device:

```bash
wpctl status
pavucontrol
```

## Audio, microphone, and PipeWire

For concepts and routing, read [PipeWire, WirePlumber, and audio
routing](../workstation/11-pipewire-wireplumber-and-audio-routing.md).

Inspect the graph, defaults, and current volume:

```bash
wpctl status
wpctl get-volume @DEFAULT_AUDIO_SINK@
wpctl get-volume @DEFAULT_AUDIO_SOURCE@
pavucontrol
```

Control the default output and microphone:

```bash
wpctl set-volume -l 1.0 @DEFAULT_AUDIO_SINK@ 5%+
wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-
wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle
wpctl set-mute @DEFAULT_AUDIO_SOURCE@ toggle
```

Set a listed PipeWire node as the deliberate default:

```bash
wpctl set-default NODE_ID
```

Diagnose the user services:

```bash
systemctl --user status pipewire.service pipewire-pulse.service wireplumber.service --no-pager
journalctl --user -b -u pipewire.service -u pipewire-pulse.service -u wireplumber.service --no-pager
```

Do not delete WirePlumber state or replace audio servers merely because one
output was selected incorrectly once.

## Battery, TLP, thermals, and brightness

For ownership and charge-threshold details, read [TLP, logind, idle handling,
and suspend](../workstation/13-tlp-logind-idle-and-suspend.md).

### Inspect power state

```bash
tlpctl list
tlpctl get
tlpctl list-holds
sudo tlp-stat -s
sudo tlp-stat -b
sudo tlp-stat -p
upower -d
sensors
brightnessctl info
```

TLP plus `tlp-pd` is the only power-profile provider. Do not install or enable
`power-profiles-daemon` or tuned beside it.

### Select a temporary profile

```bash
tlpctl performance
tlpctl balanced
tlpctl power-saver
```

Return to automatic source-based selection:

```bash
sudo tlp start
tlpctl get
```

Hold a profile only while one command runs:

```bash
tlpctl launch --profile performance --reason "local build" COMMAND
tlpctl list-holds
```

### Temporarily charge for travel

Temporarily allow the main ThinkPad battery to charge fully:

```bash
sudo tlp fullcharge BAT0
```

Restore the configured 75-80% thresholds without waiting for the next boot:

```bash
sudo tlp setcharge
sudo tlp-stat -b
```

Do not run `tlp recalibrate` as routine maintenance; it performs a deliberate
deep battery cycle and is not a battery repair command.

### Brightness

```bash
brightnessctl --list
brightnessctl info
brightnessctl set +5%
brightnessctl set 5%-
brightnessctl set 50%
```

## Lock, suspend, reboot, and power off

Lock the current Niri session:

```bash
swaylock -f
```

Inspect active inhibitors before an unexpected sleep diagnosis:

```bash
systemd-inhibit --list
loginctl session-status
```

Request supported power transitions:

```bash
systemctl suspend
systemctl reboot
systemctl poweroff
```

Save work first. `systemctl suspend` must return to the locker. Hibernation is
not configured and must not be substituted for suspend.

Inspect the chapter 18 automatic-suspend boundary:

```bash
busctl --system get-property \
  org.freedesktop.UPower \
  /org/freedesktop/UPower \
  org.freedesktop.UPower \
  OnBattery
pgrep -a -x swayidle
journalctl -b -t idle-suspend --no-pager
test -x ~/.local/bin/idle-suspend
```

The 30-minute stage requests suspend only for `b true`. Keep a long command
awake explicitly when it may otherwise run unattended on battery:

```bash
systemd-inhibit --what=sleep --mode=block \
  --why="Command must finish" COMMAND
```

This blocks the final sleep request, not the earlier lock or monitor-off
stages. Prefer AC for unattended full upgrades.

From the `niri-dotfiles` repository, confirm that the permission is portable
and not only fixed in the current working tree:

```bash
git ls-files --stage scripts/.local/bin/idle-suspend
```

The first field must be `100755`. If a Windows checkout is preparing the
correction, stage it explicitly and verify it before committing:

```bash
git add --chmod=+x scripts/.local/bin/idle-suspend
git diff --cached --summary
git ls-files --stage scripts/.local/bin/idle-suspend
```

After a suspend problem, inspect the current or previous boot as appropriate:

```bash
journalctl -b -u systemd-suspend.service --no-pager
journalctl -b -1 -u systemd-suspend.service --no-pager
journalctl -b -k --no-pager | rg -i 'suspend|resume|sleep|amdgpu|wifi|wlan'
```

## Processes and resource monitoring

```bash
btop
free -h
uptime
ps -ef
pgrep -a -x PROCESS_NAME
ps -fp PID
lsof FILE_OR_DIRECTORY
```

Ask a process to terminate normally:

```bash
kill -TERM PID
```

Use `kill -KILL PID` only when the exact process is confirmed and cannot
respond to normal termination; it cannot save state or clean up.

## systemd services, user services, timers, and logs

For the complete state model, read [Systemd units, activation, and the
journal](../foundations/03-systemd-units-activation-and-journal.md).

### Inspect system state

```bash
systemctl is-system-running
systemctl --failed --no-pager
systemctl --user --failed --no-pager
systemctl list-timers --all --no-pager
```

### Inspect or control one system unit

```bash
systemctl status NAME.service --no-pager
systemctl is-enabled NAME.service
systemctl is-active NAME.service
systemctl cat NAME.service
sudo systemctl start NAME.service
sudo systemctl stop NAME.service
sudo systemctl restart NAME.service
sudo systemctl enable --now NAME.service
sudo systemctl disable --now NAME.service
```

`enable` controls future activation; `start` controls the current runtime.
Use `disable --now` only when both changes are intended.

### Inspect or control one user unit

```bash
systemctl --user status NAME.service --no-pager
systemctl --user is-enabled NAME.service
systemctl --user is-active NAME.service
systemctl --user cat NAME.service
systemctl --user restart NAME.service
```

Do not add `sudo` to `systemctl --user`; that would address the wrong user
manager.

### Read the journal

```bash
journalctl -b --no-pager
journalctl -b -p warning..alert --no-pager
journalctl --user -b -p warning..alert --no-pager
journalctl -b -u NAME.service --no-pager
journalctl --user -b -u NAME.service --no-pager
journalctl -b -k --no-pager
journalctl -b -1 --no-pager
journalctl -f
```

Bound large queries when possible:

```bash
journalctl -b --since "30 minutes ago" --no-pager
journalctl -b -u NAME.service --since "30 minutes ago" --no-pager
```

A warning is evidence to interpret, not automatically a fault. Compare it with
the unit state, timestamp, observed behavior, and earlier boots.

## Firewall and exposed services

For the security model, read [Firewalld, nftables, zones, and host
exposure](../networking/09-firewalld-nftables-zones-and-host-exposure.md).

Read-only daily checks:

```bash
firewall-cmd --state
firewall-cmd --get-default-zone
firewall-cmd --get-active-zones
firewall-cmd --list-all
sudo ss -tulpn
```

The firewall and the list of listening sockets answer different questions.
Do not add a service or port permanently merely to silence a connectivity
problem; identify which program needs which exposure and on which zone first.

Confirm that the intentionally unused SSH server remains disabled:

```bash
systemctl is-enabled sshd.service
systemctl is-active sshd.service
```

## Firmware updates

Firmware metadata refreshes automatically, while installation remains manual:

```bash
fwupdmgr get-devices
fwupdmgr refresh
fwupdmgr get-updates
fwupdmgr check-reboot-needed
```

Before accepting an offered update, connect the charger, read the release
details, confirm battery state, and verify the owner-signed updater:

```bash
upower -d
sudo sbctl status
sudo sbctl verify
sudo sbctl verify /usr/lib/fwupd/efi/fwupdx64.efi.signed
```

Apply only a reviewed offered update:

```bash
sudo fwupdmgr update
```

After its required reboot:

```bash
fwupdmgr get-history
sudo sbctl verify
```

Never interrupt a firmware write or apply firmware intended for another model.

## Boot, encryption, mounts, and Secure Boot

These are read-only health checks for normal operation:

```bash
bootctl --esp-path=/boot status
bootctl --esp-path=/boot list
sudo sbctl status
sudo sbctl verify
sudo cryptsetup status cryptlvm
findmnt / /home /boot
swapon --show
```

The live installed system must show `cryptlvm` active. A different result from
inside an installation environment, chroot, or against a mistyped mapping name
must be interpreted in that context.

Regenerate UKIs only after an intentional kernel, initramfs, command-line, or
preset change:

```bash
sudo mkinitcpio -P
sudo sbctl verify
```

Do not run regeneration as a speculative fix. If it fails, do not reboot until
both normal and fallback UKIs plus systemd-boot verify correctly.

After post-install chapter 19, inspect the two Plymouth presentation policies:

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
cat /etc/kernel/cmdline-fallback
cat /etc/mkinitcpio.d/linux.preset
plymouth-set-default-theme
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
```

The normal UKI must include `quiet splash`; the fallback must include neither
and its preset must skip both `autodetect` and `plymouth`. During a normal
Plymouth boot, press `Esc` to reveal detailed messages. If the graphical LUKS
request is invisible or unusable, select the textual fallback UKI at the next
systemd-boot menu instead of typing the passphrase blindly.

After post-install chapter 20, inspect the additional TPM and signed-PCR
policy without changing it:

```bash
systemd-analyze has-tpm2
systemd-analyze pcrs 7 11
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo ukify inspect --section=.pcrsig:text --section=.pcrpkey:text /boot/EFI/Linux/arch-linux.efi
sudo test -s /run/systemd/tpm2-pcr-signature.json
sudo test -s /run/systemd/tpm2-pcr-public-key.pem
sudo sbctl verify
```

The normal UKI requests `tpm2-device=auto` and accepts the unique TPM PIN.
Fallback omits that option and continues to require the strong LUKS
passphrase or recovery key. After a boot-related package update, regenerate
and inspect both UKIs as documented in chapter 20; a valid newly signed PCR 11
value does not require routine TPM reenrollment. Never clear the TPM or wipe a
password/recovery slot to fix a failed PIN path.

## Restic backup routine

For disk identification, LUKS handling, recovery material, and restore drills,
use the complete post-install backup chapter. The commands below assume the
verified encrypted disk is already unlocked and mounted at the canonical path.

Define a session-local repository path without storing the password:

```bash
restic_repo=/run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad
test -d "$restic_repo"
```

Create a manual user-data snapshot:

```bash
restic --repo "$restic_repo" backup /home/neon --one-file-system --exclude '/home/neon/.cache' --exclude '/home/neon/.local/share/Trash' --tag manual
```

Inspect snapshots, the latest snapshot, and repository integrity:

```bash
restic --repo "$restic_repo" snapshots
restic --repo "$restic_repo" stats latest
restic --repo "$restic_repo" check --read-data-subset=5%
```

List a snapshot before restoring:

```bash
restic --repo "$restic_repo" ls latest
```

Restore into a new empty target, never over the live home directory:

```bash
mkdir -m 0700 /home/neon/restic-restore-test
restic --repo "$restic_repo" restore latest --target /home/neon/restic-restore-test --include /home/neon/PATH_TO_RESTORE
```

Compare the restored content, then move the test directory to Trash through
Nautilus or `gio trash` when it is no longer needed. Finish every backup by
using the chapter 12 sequence to unmount, lock, and power off the exact disk.
Never unplug it merely because Restic returned to the prompt.

## Git and project repositories

For the complete model and recovery workflow, read [Git, GitHub, SSH
authentication, and repository
workflow](../git-and-github/22-git-github-ssh-and-repository-workflow.md).

### Inspect before changing anything

```bash
git status --short --branch
git diff
git diff --staged
git log --oneline --decorate --graph -n 15
git remote -v
```

### Update a clean branch

```bash
git status --short --branch
git fetch --prune origin
git pull --ff-only
```

Do not pull over unexplained local changes. `--ff-only` stops instead of
creating an accidental merge commit.

### Review and publish a change

```bash
git diff
git add -p
git diff --staged
git commit
git push
```

Prefer deliberate paths or `git add -p` over `git add .`. Confirm that no
secret, generated state, private key, token, or Wi-Fi credential is staged.

### Branches and tags

```bash
git branch --show-current
git branch --all
git switch main
git switch -c TOPIC_BRANCH
git tag --list
git describe --tags --always
```

Never use `git reset --hard`, force-push, or delete a branch as a routine way
to make a confusing state disappear.

### Check the four project clones

```bash
git -C ~/Projects/CycloniteRDX/arch-linux-runbook status --short --branch
git -C ~/Projects/CycloniteRDX/arch-linux-post-install status --short --branch
git -C ~/Projects/CycloniteRDX/arch-linux-handbook status --short --branch
git -C ~/Projects/CycloniteRDX/niri-dotfiles status --short --branch
```

## Niri, desktop components, and dotfiles

### Useful Niri commands

```bash
niri validate
niri msg outputs
niri msg windows
niri msg layers
niri msg action quit
```

`niri msg action quit` ends the graphical session; it does not power off the
computer. Validate configuration before logging out of a working session.

Inspect the selected session components:

```bash
pgrep -a -x waybar
pgrep -a -x mako
pgrep -a -x swaybg
pgrep -a -x swayidle
pgrep -a -x swaylock
busctl --user status org.freedesktop.Notifications
systemctl --user --failed --no-pager
```

`swaylock` is normally absent while the session is unlocked. Each other
selected role should have exactly one owner.

### Inspect Qt 6 appearance

```bash
printf 'QT_QPA_PLATFORMTHEME=%s\n' "$QT_QPA_PLATFORMTHEME"
printenv QT_QPA_PLATFORM || true
printenv QT_STYLE_OVERRIDE || true
pacman -Q qt6ct qt6-base qt6-svg
readlink -f ~/.config/qt6ct/qt6ct.conf
readlink -f ~/.config/qt6ct/colors/midnight-circuit.conf
grep -E '^(color_scheme_path|custom_palette|icon_theme|standard_dialogs|style)=' \
  ~/.config/qt6ct/qt6ct.conf
```

The selected state is `QT_QPA_PLATFORMTHEME=qt6ct`, Fusion, Papirus Dark, the
Midnight Circuit scheme, and `xdgdesktopportal`. `QT_QPA_PLATFORM` and
`QT_STYLE_OVERRIDE` remain unset globally.

Test one application's backend without changing the session:

```bash
QT_QPA_PLATFORM=wayland QT_APPLICATION
QT_QPA_PLATFORM=xcb QT_APPLICATION
```

These are diagnostic prefixes for one process. Use only the command that
matches the question being tested; do not export either value globally. The
normal qt6ct editor can modify its tracked configuration through the Stow link,
so use chapter 17's temporary inspection procedure unless the change is
intentional, then review `git diff`.

### Portable keyboard shortcuts

| Shortcut | Action |
| --- | --- |
| `Super+Enter` | Open Kitty. |
| `Super+D` | Open Fuzzel. |
| `Super+O` | Toggle the Niri overview. |
| `Super+Q` | Close the focused window. |
| `Super+Alt+L` | Lock with swaylock. |
| `Super+Shift+/` | Show Niri's important-shortcut overlay. |
| `Super+Page Up/Down` | Change workspace. |
| `Super+Ctrl+Page Up/Down` | Move the focused column to another workspace. |
| `Super+R` | Cycle column widths. |
| `Super+F` | Maximize the focused column. |
| `Super+Shift+F` | Toggle fullscreen. |
| `Super+V` | Toggle floating. |
| `Super+W` | Toggle tabbed display for the column. |
| `Print` | Interactive screenshot. |
| `Ctrl+Print` | Screenshot the focused output. |
| `Alt+Print` | Screenshot the focused window. |
| `Super+Shift+E` | Exit Niri through its confirmation dialog. |

### Clipboard, notifications, and default applications

```bash
printf '%s' 'TEXT' | wl-copy
wl-paste
wl-paste --primary
notify-send "Title" "Message"
xdg-open PATH_OR_URL
xdg-mime query default MIME_TYPE
gio mime MIME_TYPE
```

### Reconcile dotfiles safely

From `~/Projects/CycloniteRDX/niri-dotfiles`, preview before changing links:

```bash
git status --short --branch
stow --simulate --verbose --no-folding --target="$HOME" niri autostart mimeapps waybar fuzzel mako wallpapers swaylock kitty theme qt6ct scripts
niri validate --config niri/.config/niri/config.kdl
```

After a reviewed repository change, reconcile the deployed links:

```bash
stow --restow --verbose --no-folding --target="$HOME" niri autostart mimeapps waybar fuzzel mako wallpapers swaylock kitty theme qt6ct scripts
niri validate
```

Stop on a Stow conflict. Do not use `--adopt` as a shortcut: it can import an
unreviewed live file into the repository.

## Practical maintenance cadence

### Before every system upgrade

1. Read new Arch news.
2. Ensure there is enough time to recover from an unexpected issue.
3. Run `checkupdates` if a preview is useful.
4. Create a current backup before a large or risky transaction.
5. Run `sudo pacman -Syu` and read all output.
6. Run `sudo pacdiff --output` and resolve every reported file.
7. Inspect failed units and new journal errors.
8. Verify Secure Boot artifacts when boot-related packages changed.

### Weekly or after important work

```bash
systemctl --failed --no-pager
systemctl --user --failed --no-pager
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad snapshots
```

Create a Restic snapshot after irreplaceable work. Confirm that the automated
cache, TRIM, and firmware-metadata timers remain scheduled:

```bash
systemctl list-timers --all paccache.timer fstrim.timer fwupd-refresh.timer --no-pager
```

### Monthly or when disk space changes unexpectedly

```bash
df -hT
gio trash --list
sudo paccache --dryrun
pacman -Qdt
pacman -Qm
journalctl -p warning..alert --since "30 days ago" --no-pager
fwupdmgr get-updates
```

Review results; do not turn this inspection list into an automatic deletion
script.

### After security, boot, or storage changes

- Re-run the relevant runbook or post-install verification.
- Verify both UKIs and systemd-boot with `sbctl verify`.
- Boot and test the fallback UKI when the boot path changed.
- Refresh the encrypted recovery bundle after LUKS keyslot, LUKS token, or
  Secure Boot key changes.
- After TPM enrollment, confirm both UKIs still contain valid signed-PCR
  sections and test normal PIN plus textual passphrase boots.
- Repeat the recovery-media drill after a material partition, encryption,
  boot-manager, or initramfs redesign.

## Commands intentionally absent from the routine

These are not everyday fixes:

- `pacman -Sy` without `-u`;
- `pacman -Scc` for normal cache maintenance;
- forced package removal with `-Rdd`;
- unreviewed `--overwrite`, `--nodeps`, or `--noconfirm`;
- piping orphan queries directly into removal;
- `rfkill unblock all`;
- forced unmounts or disconnecting a busy backup disk;
- `systemctl hibernate` on this no-hibernation profile;
- `sudo pip` against the Arch-managed Python installation;
- recursive forced deletion from an unverified path;
- `git reset --hard` or force-push as generic conflict recovery;
- speculative `mkinitcpio -P`, key regeneration, or Secure Boot disablement.

When one of these seems necessary, use the detailed guide to identify the
actual fault and construct a bounded recovery operation.

## Sources

- [Arch manual: pacman(8)](https://man.archlinux.org/man/pacman.8.en)
- [Arch manual: checkupdates(8)](https://man.archlinux.org/man/checkupdates.8.en)
- [Arch manual: paccache(8)](https://man.archlinux.org/man/paccache.8.en)
- [Arch manual: pacdiff(8)](https://man.archlinux.org/man/pacdiff.8.en)
- [Paru upstream manual](https://github.com/Morganamilo/paru/blob/master/man/paru.8)
- [Arch manual: gio(1)](https://man.archlinux.org/man/gio.1.en)
- [NetworkManager: nmcli examples](https://networkmanager.dev/docs/api/latest/nmcli-examples.html)
- [Arch manual: bluetoothctl(1)](https://man.archlinux.org/man/bluetoothctl.1.en)
- [WirePlumber: wpctl](https://pipewire.pages.freedesktop.org/wireplumber/man/wpctl.html)
- [TLP: tlpctl](https://linrunner.de/tlp/usage/tlpctl.html)
- [TLP: command-line usage](https://linrunner.de/tlp/usage/tlp.html)
- [Arch manual: systemctl(1)](https://man.archlinux.org/man/systemctl.1.en)
- [Arch manual: journalctl(1)](https://man.archlinux.org/man/journalctl.1.en)
- [Arch manual: udisksctl(1)](https://man.archlinux.org/man/udisksctl.1.en)
- [Niri documentation](https://niri-wm.github.io/niri/)
- [Arch package: qt6ct](https://archlinux.org/packages/extra/x86_64/qt6ct/)
- [Niri environment configuration](https://niri-wm.github.io/niri/Configuration:-Miscellaneous.html#environment)
- [Restic documentation](https://restic.readthedocs.io/en/latest/)
