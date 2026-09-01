# Printing, scanning, and peripheral integration

## Purpose and scope

A desktop application makes printing look like one operation: select a device
and press **Print**. Linux may actually be coordinating a document format, a
local scheduler, a queue, filters, a transport protocol, network discovery,
authorization, and the printer itself. Scanning and general USB devices have
similar boundaries.

This guide explains:

- the difference between a physical printer and a CUPS queue;
- why IPP Everywhere is preferred over model-specific printer drivers;
- local CUPS operation, queue administration, jobs, and authorization;
- known printer addresses versus DNS-SD discovery;
- why accepting print jobs from the network is different from sending them;
- modern IPP-over-USB devices and the role of `ipp-usb`;
- SANE, eSCL/AirScan, WSD, scanner discovery, and graphical frontends;
- kernel enumeration, udev identity, permissions, exclusive ownership, and
  power policy for other peripherals;
- a layer-by-layer diagnostic and recovery method.

The post-install repository remains the executable procedure. This guide does
not install packages, add a printer, change CUPS configuration, enable Avahi,
open firewalld services, share a printer or scanner, add the user to a group,
or create udev rules merely by being copied.

Printer sharing, scanner sharing, vendor-specific proprietary drivers,
enterprise print accounting, smart-card devices, USBGuard policy, and virtual
PDF printers are outside the canonical laptop baseline. They must be designed
for an identified device or role instead of enabled speculatively.

## Current project contract

Printing is optional because neither ThinkPad needs a permanent print server
role. When printing is required, the selected baseline is:

| Responsibility | Selected component | State |
| --- | --- | --- |
| Print client and scheduler | CUPS | Installed conditionally |
| Document conversion | `cups-filters` | Installed with the print stack |
| Graphical administration | `system-config-printer` | Installed conditionally |
| Graphical authorization bridge | `cups-pk-helper` | Installed conditionally |
| Scheduler activation | `cups.socket` | Enabled only with the print stack |
| Network printer protocol | IPP Everywhere | Preferred |
| USB IPP bridge | `ipp-usb` | Only for a compatible identified device |
| Scanner API and tools | SANE | Only when a scanner is identified |
| Driverless network scanning | `sane-airscan` | Preferred for eSCL or WSD |
| Graphical scanning | Simple Scan | Optional frontend |
| Automatic network discovery | Avahi/DNS-SD | Off unless deliberately selected |
| Automatic remote queues | `cups-browsed` | Off unless deliberately selected |
| Printer or scanner sharing | None | Off |

The firewall remains managed by firewalld over nftables, NetworkManager
continues to own connection-to-zone assignment, and `public` remains the
default zone. A client connecting to a known printer address does not require
opening an inbound print-server port on the laptop.

The project does not create a broad polkit rule for printing. The existing
graphical polkit authentication agent can present authorization requests from
`cups-pk-helper`; CUPS also has its own `@SYSTEM` administrative identity.
Those mechanisms must be inspected independently.

Unless a section explicitly says otherwise, command examples in this article
run in Bash on the installed Arch system. The delivery instructions use
PowerShell separately on Windows.

## The printing path

The useful model is a sequence of ownership boundaries:

```mermaid
flowchart LR
    A["Application"] --> B["CUPS client"]
    B --> C["Scheduler and queue"]
    C --> D["Filter or driverless conversion"]
    D --> E["IPP, USB, or legacy backend"]
    E --> F["Physical printer"]
```

| Layer | Question it answers | Typical inspection |
| --- | --- | --- |
| Application | Did the application create and submit a job? | Application print dialog and logs |
| CUPS client | Which destination and options were selected? | `lpstat`, `lpoptions` |
| Scheduler | Is the queue accepting and processing jobs? | `lpstat -t`, CUPS journal |
| Conversion | Can the document become a printer-supported format? | Job state and filter errors |
| Backend | Which URI or local interface carries the job? | `lpstat -v`, `lpinfo -v` |
| Device | Is the printer ready, supplied, and error-free? | Printer panel or web interface |

A job visible in CUPS proves that the application reached the scheduler. It
does not prove that conversion succeeded, that the network path is reachable,
or that the physical printer has paper.

### Printer, destination, and queue

The physical printer is hardware. A CUPS **destination** is a named printer
queue or a class of queues. A queue records a device URI, capabilities,
defaults, job state, and policy. It can remain present while the hardware is
offline.

Consequently:

- deleting a queue does not remove a physical printer;
- unplugging a printer does not delete its queue;
- two queues can refer to the same hardware with different defaults;
- the application normally prints to the destination name, not directly to a
  USB node or IP address.

### Scheduler, filters, and backends

`cupsd` implements the local print service and IPP scheduler. It accepts jobs,
applies destination policy, starts conversion filters when needed, and invokes
the backend represented by the queue URI.

A **filter** transforms document data. A **backend** transports the resulting
job. They are different responsibilities. For example, `ipp://` selects the
IPP backend, while `-m everywhere` tells CUPS to query an IPP Everywhere
printer and use its advertised capabilities.

## Driverless printing first

Modern driverless printing does not mean that the computer performs no work.
It means the printer advertises capabilities and accepts standardized formats
and operations, avoiding a model-specific PPD and arbitrary vendor filter.

For this project, the preference order is:

1. IPPS or IPP Everywhere at a known address or stable hostname;
2. an automatically discovered IPP Everywhere destination on a deliberately
   trusted LAN;
3. IPP-over-USB through `ipp-usb` for a compatible USB device;
4. a model-specific open-source driver from the official repositories;
5. a reviewed vendor or AUR driver only when the hardware requires it.

PPD-based drivers and non-`everywhere` CUPS models are deprecated upstream.
They can still be necessary for older hardware, but installing every driver
collection “just in case” increases code, filters, maintenance, and ambiguity.

### Compare the common paths

| Device path | Discovery | Main advantage | Main trade-off |
| --- | --- | --- | --- |
| Known `ipp://` or `ipps://` URI | Not required | Small, predictable network surface | Address or hostname must remain stable |
| DNS-SD IPP destination | mDNS/Avahi | Convenient on a trusted local LAN | Adds multicast discovery and network dependence |
| IPP-over-USB | `ipp-usb` advertises a local service | Driverless printing and often scanning over one USB cable | Claims the compatible USB interface; conflicts with some classic drivers |
| Classic USB backend | `usb://` URI reported by CUPS | Supports older USB printers | Usually requires a model-specific driver/PPD |
| Legacy network protocol | LPD, AppSocket, or vendor path | Supports older print servers | Weaker capability discovery and more manual configuration |

Use the exact URI reported by CUPS or the device documentation. Do not invent
a `usb://` URI, guess a queue path, or use `/dev/lp0` as a persistent identity.

## Packages are roles, not one indivisible stack

The post-install procedure conditionally installs:

```text
cups
cups-filters
cups-pk-helper
system-config-printer
```

Their roles are distinct:

| Package | Role |
| --- | --- |
| `cups` | Scheduler, IPP client/server, CLI tools, backends, and systemd units |
| `cups-filters` | Conversion support needed by some printer paths |
| `cups-pk-helper` | D-Bus helper allowing graphical tools to request polkit-authorized administration |
| `system-config-printer` | GTK queue discovery and administration frontend |

`cups-pk-helper` is not a print driver and the polkit agent is not the helper.
The helper requests authorization; polkit evaluates it; the graphical agent
may collect the password.

Other packages are selected only after identifying the device:

| Need | Candidate | Selection rule |
| --- | --- | --- |
| Modern USB IPP device | `ipp-usb` | Device explicitly supports IPP-over-USB |
| Driverless scanning | `sane`, `sane-airscan` | Multifunction device exposes eSCL or WSD |
| Simple GTK scanner UI | `simple-scan` | A graphical frontend is desired |
| Broad legacy printer support | `gutenprint` or a model package | Compatibility list matches the exact model |
| HP-specific legacy features | `hplip` | Exact model and required function justify it |
| Persistent automatic remote queues | `cups-browsed` | A real multi-printer discovery workflow needs it |

Do not treat a manufacturer name as sufficient evidence. One model family may
support IPP Everywhere while another needs a legacy driver, and a
multifunction device can use different protocols for printing and scanning.

## CUPS activation and local scope

The project enables `cups.socket`, not an unconditional permanent print
server. Socket activation allows systemd to start the scheduler when a local
client connects.

Inspect all related units together:

```bash
systemctl is-enabled cups.socket cups.service cups.path
systemctl is-active cups.socket cups.service cups.path
systemctl status cups.socket cups.service cups.path
systemctl list-dependencies cups.socket
```

The states are not contradictory when the socket is active and the service is
currently inactive. The service can start on demand. Guide 03 explains the
general activation model.

Inspect the effective server state without changing it:

```bash
cupsctl
ss -lntup
sudo cupsd -t
sudo systemd-analyze cat-config cups/cupsd.conf
sudo systemd-analyze cat-config cups/cups-files.conf
```

`cupsd -t` validates CUPS configuration syntax. It does not prove that a
printer is reachable or a policy is desirable.

The local web interface is normally available at:

```text
http://localhost:631/
```

The presence of a web interface on loopback is not equivalent to exposing it
on the LAN. Listener addresses, CUPS access rules, queue sharing, service
discovery, and the firewall all contribute to remote exposure.

### Keep sharing disabled

The selected laptop policy is:

```bash
sudo cupsctl --no-share-printers --no-remote-admin --no-remote-any
```

This is a state-changing command and belongs in the operational procedure only
when CUPS is installed. Afterward, inspect rather than infer:

```bash
cupsctl
ss -lntup
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

Upstream `lpadmin` documents `printer-is-shared=true` as the destination
default. Global CUPS sharing controls still govern actual publication, but the
project sets each local queue explicitly to unshared so both layers record the
same intent.

## Discover and create a queue deliberately

Begin with read-only inspection:

```bash
lpstat -t
lpinfo -v
lpinfo -m | grep -F 'IPP Everywhere'
```

`lpinfo -v` reports available backend schemes and discovered device URIs. A
reported destination is not automatically trusted; confirm its model,
location, address, and ownership.

The graphical route is `system-config-printer`. The CLI equivalent for a
known driverless IPP printer has this shape:

```bash
sudo lpadmin \
    -p office_printer \
    -E \
    -v 'ipp://PRINTER_ADDRESS/ipp/print' \
    -m everywhere \
    -o printer-is-shared=false
```

`PRINTER_ADDRESS` and the resource path are placeholders. Confirm them using
the printer's documentation or web interface. Quoting the URI also prevents
shell interpretation of characters that can occur in it.

Verify the exact destination before setting it as default:

```bash
lpstat -p office_printer -l
lpstat -v office_printer
lpoptions -p office_printer -l
```

The system-wide default can then be selected administratively:

```bash
sudo lpadmin -d office_printer
```

A user-specific default can instead be selected with `lpoptions -d`; know
which scope is intended before troubleshooting two different defaults.

### Test through the same layers as an application

First test scheduler reachability and destination state:

```bash
lpstat -r
lpstat -d
lpstat -t
```

Then submit a known small local document:

```bash
lp -d office_printer /usr/share/cups/data/testprint
lpstat -W not-completed -o office_printer
```

Record the returned job identifier. It makes cancellation and log correlation
precise:

```bash
cancel JOB_ID
journalctl -u cups.service --since '-10 min' --no-pager
```

Do not cancel every user's jobs or purge an entire queue merely because one
test failed.

## Jobs, acceptance, and enabled state

A destination has several independent states:

| State | Meaning |
| --- | --- |
| Accepting jobs | New submissions can enter the queue |
| Enabled | CUPS may process queued jobs |
| Idle | No job is currently printing |
| Processing | A job is being converted or sent |
| Stopped | An error policy or administrator stopped processing |
| Device offline | Backend cannot currently reach the physical device |

Inspect before applying a remedy:

```bash
lpstat -p office_printer -l
lpstat -W all -o office_printer
```

Commands such as `cupsenable`, `cupsdisable`, `cupsaccept`, and `cupsreject`
change different controls. “Enabled” does not imply “accepting,” and clearing
a physical paper jam does not necessarily resume a queue stopped by policy.

## Authorization: submission is not administration

Ordinary users normally submit and manage their own jobs. Creating queues,
changing server policy, or cancelling another user's job is administrative.
Do not solve an administrative prompt by running the graphical desktop as
root.

There are two relevant authorization paths:

| Path | Purpose | Source of policy |
| --- | --- | --- |
| Native CUPS administration | Authenticates operations protected by CUPS policy | `cupsd.conf` policies and CUPS `@SYSTEM` |
| Graphical helper | Lets a GUI request privileged queue changes | `cups-pk-helper` plus polkit |

CUPS expands `@SYSTEM` using `SystemGroup` in `cups-files.conf`. Inspect the
effective setting rather than assuming that `wheel`, `sys`, or another group
is authoritative on every distribution or future version:

```bash
sudo systemd-analyze cat-config cups/cups-files.conf
sudo grep -Rns --include='*.conf' '^[[:space:]]*SystemGroup' /etc/cups /usr/share/cups 2>/dev/null
id
```

Do not add `neon` to `cups`, `lp`, `sys`, or `wheel` until a specific
documented authorization boundary requires it. Service accounts, device
groups, and CUPS administrative groups are separate roles, and their exact
use has changed across CUPS and distribution versions. Membership is not a
generic “make printing work” switch.

For graphical administration, inspect the helper and polkit decision path:

```bash
pacman -Q cups-pk-helper polkit
busctl --system list | grep -E 'cups|PolicyKit'
journalctl -b _COMM=polkitd --no-pager
```

The existing Niri session must have its polkit authentication agent running
for an interactive graphical prompt. Guide 10 explains daemon, rules, agent,
and portal boundaries. This project does not add a blanket `wheel` rule that
silently approves every printing action.

## Network printers: connection and discovery are separate

Knowing a printer's IPP URI lets the laptop make an outbound connection. DNS
Service Discovery lets the laptop learn that URI automatically using local
multicast. A successful connection does not require discovery, and discovery
does not prove the printer accepts jobs.

The project prefers a known address combined with a DHCP reservation or a
stable local hostname. This avoids making every network joined by the laptop
a discovery domain.

Before considering discovery, inspect the current connection and zone:

```bash
nmcli -f NAME,UUID,TYPE,DEVICE,ZONE connection show --active
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

Also inspect firewalld's packaged service definitions instead of memorizing
ports:

```bash
sudo firewall-cmd --info-service=mdns
sudo firewall-cmd --info-service=ipp-client
sudo firewall-cmd --info-service=ipp
sudo firewall-cmd --info-service=ws-discovery-client
sudo firewall-cmd --info-service=sane
```

The names express different roles:

| Service definition | Typical role |
| --- | --- |
| `mdns` | DNS-SD/mDNS discovery on the local link |
| `ipp-client` | Client-side IPP-related traffic expected by that definition |
| `ipp` | Accepting inbound IPP as a server |
| `ws-discovery-client` | Discovering WSD devices |
| `sane` | Serving classic SANE network access |

Printing **to** a known network printer does not justify adding the inbound
`ipp` service to the laptop. Likewise, using a remote scanner does not justify
running or exposing `saned` locally.

### A bounded discovery test

Automatic discovery is allowed only on a deliberately trusted current LAN.
If mDNS is genuinely required, test a runtime rule with a timeout rather than
making it permanent immediately:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-service=mdns --timeout=10m
lpinfo -v
airscan-discover
```

`public` is the current project default, but the first command must confirm
the active zone. The timeout limits the experiment. A reload also discards
runtime-only changes, so do not run `firewall-cmd --runtime-to-permanent` as a
shortcut: it would copy every unrelated runtime experiment.

If discovery works, decide whether convenience justifies a permanent rule on
that connection's deliberately assigned zone. The safer long-term answer may
still be to record a known URI and leave multicast discovery off.

### Avahi and `cups-browsed`

Avahi implements mDNS/DNS-SD discovery and publication. Having its libraries
or package installed does not mean `avahi-daemon.service` must be enabled.

`cups-browsed` is a separate daemon that can create and maintain local queues
for remote CUPS queues and network printers. It is not required for a single
known IPP destination. Before enabling it, define:

- which networks may supply destinations;
- whether transient queues are acceptable;
- how duplicate or renamed queues will be handled;
- whether the laptop should retain that behavior on public Wi-Fi;
- which discovery and firewall rules are necessary.

The canonical workstation leaves both automatic services off until a real
environment needs them.

## Printer sharing is a server role

Receiving jobs from another machine is not a small extension of local
printing. It changes the laptop into a server and requires a complete design:

- CUPS listening and access policy;
- per-queue sharing state;
- authentication and possibly TLS certificates;
- firewalld inbound IPP scope;
- DNS-SD advertisement if discovery is desired;
- client isolation and trust on the router or access point;
- job, spool, and log privacy;
- availability while the laptop sleeps or changes networks.

Do not enable sharing with a single copied command. Upstream documents
`cupsctl --share-printers` and `printer-is-shared=true`, but the project uses
their disabled equivalents. A portable laptop that suspends and joins several
networks is usually a poor print server.

## IPP-over-USB

Some modern USB printers expose IPP, eSCL scanning, and a device web interface
over a USB transport. Applications expect HTTP/IPP, while the physical link is
USB. `ipp-usb` acts as a local reverse proxy between those layers.

```mermaid
flowchart LR
    A["CUPS or SANE client"] --> B["Local HTTP or IPP endpoint"]
    B --> C["ipp-usb"]
    C --> D["IPP-over-USB interface"]
    D --> E["Multifunction device"]
```

This path is attractive because the same driverless protocols used over the
network can work over a cable. It also introduces an ownership decision:
`ipp-usb` claims the compatible USB interface. A classic CUPS USB backend or
vendor driver may then stop seeing the same interface.

Therefore:

1. confirm the device advertises IPP-over-USB;
2. choose either the IPP-over-USB path or the classic vendor path;
3. inspect `ipp-usb.service`, its journal, and discovered local URI;
4. test both printing and scanning before declaring the multifunction device
   complete;
5. do not install several competing driver stacks simultaneously.

Inspect a selected installation with:

```bash
systemctl status ipp-usb.service
journalctl -u ipp-usb.service -b --no-pager
lpinfo -v
scanimage -L
```

The upstream default is intended to expose the proxy locally. Do not change
its interface from loopback to all interfaces merely to make discovery easier;
that can publish the USB device to the LAN and creates a server role that this
project has not selected.

## Legacy printers and vendor software

When IPP Everywhere and IPP-over-USB are unavailable, identify the exact
model, hardware revision, connection type, and required features before
choosing software.

Use this order:

1. search the Arch official repositories;
2. read the driver's supported-model list and upstream status;
3. inspect package contents, dependencies, services, udev rules, filters, and
   network listeners;
4. prefer a narrow model-specific package over multiple overlapping suites;
5. review PKGBUILDs and source origins before considering the AUR;
6. keep a tested removal and queue-recreation path.

Do not download an arbitrary binary installer from a search result. Vendor
installers may overwrite files outside pacman's database, add broad udev
rules, start network daemons, or install filters that remain after the queue is
removed.

If a legacy queue is necessary, record why `-m everywhere` failed and the
exact model identifier selected from `lpinfo -m`. This evidence makes a future
driverless migration possible.

## The scanning path

SANE is an API and backend framework. It is not one daemon that must always be
enabled.

```mermaid
flowchart LR
    A["Simple Scan or scanimage"] --> B["SANE API"]
    B --> C["Device backend"]
    C --> D["USB, eSCL, WSD, or network"]
    D --> E["Scanner"]
```

| Layer | Example | Responsibility |
| --- | --- | --- |
| Frontend | Simple Scan or `scanimage` | User interface and scan request |
| API/router | SANE DLL layer | Loads and exposes configured backends |
| Backend | `sane-airscan` or a model backend | Implements device protocol |
| Discovery | DNS-SD, WSD, or a manual URL | Finds a network service |
| Transport | USB or IP network | Carries commands and image data |

`saned` is a server for exporting SANE devices to remote clients. It is not
required for a local USB scanner or for this laptop to use a remote eSCL
scanner. Keep `saned.socket` disabled unless scanner sharing becomes an
explicit, secured requirement.

### Driverless eSCL and WSD scanning

`sane-airscan` supports eSCL, also called AirScan, and WSD/WS-Scan. Begin with:

```bash
scanimage -L
airscan-discover
```

`airscan-discover` uses Avahi for DNS-SD devices and its own WS-Discovery
implementation for WSD devices. It emits a configuration fragment; review it
before copying anything into `/etc/sane.d/airscan.conf`.

When automatic discovery is unreliable or undesirable, configure only a
known device. The shape documented by `sane-airscan(5)` is:

```ini
[devices]
"Office scanner" = http://SCANNER_ADDRESS:PORT/eSCL, eSCL
```

The address, port, path, and protocol are placeholders. Obtain them from
device documentation or a reviewed discovery result. A drop-in under
`/etc/sane.d/airscan.d/` can keep local policy separate when supported by the
installed version; guide 01 explains why program-specific precedence must
always be verified.

The backend can also disable automatic discovery when only manual devices are
desired:

```ini
[options]
discovery = disable
ws-discovery = off
```

Do not copy those settings before confirming that they match the intended
workflow. They are a policy choice, not a universal repair.

### Verify the backend before blaming the GUI

List devices and record the complete backend name:

```bash
scanimage -L
```

Inspect the options of one exact device:

```bash
scanimage --help --device-name 'BACKEND_DEVICE_NAME'
scanimage --test --device-name 'BACKEND_DEVICE_NAME'
```

Then acquire a small test to a deliberate path:

```bash
scanimage \
    --device-name 'BACKEND_DEVICE_NAME' \
    --format=png \
    --output-file="$HOME/Downloads/scanner-test.png"
```

The backend name is a placeholder returned by `scanimage -L`. If the CLI works
but Simple Scan does not, the hardware and backend path are already proven;
focus on the frontend session, sandbox, or saved state.

For a USB scanner, `sane-find-scanner` can help prove low-level enumeration,
but finding hardware there does not prove that a SANE backend supports it.

## General peripheral integration

Printers and scanners illustrate a broader rule: solve the first failing
layer, not the most visible symptom.

| Boundary | Evidence | Typical failure |
| --- | --- | --- |
| Physical | Power, cable, port, dock, device mode | Charge-only cable, weak hub, wrong mode |
| Kernel enumeration | `lsusb`, kernel journal | Missing driver, disconnect loop, USB error |
| udev identity | Properties, symlinks, tags | Wrong rule or unstable name |
| Access | Owner, group, ACL, active seat | User cannot open device |
| Protocol/backend | CUPS, SANE, V4L2, HID, serial | Unsupported protocol or missing backend |
| Session/application | Portal, GUI, exclusive owner | Wrong session, stale state, competing process |
| Power policy | TLP and autosuspend | Device disappears after idle or suspend |

### Start at the physical boundary

Check:

- device power and error indicators;
- a known data-capable cable, not only a charging cable;
- a direct laptop port before a dock or unpowered hub;
- the device's selected USB/network mode;
- whether failure follows the device, cable, port, or laptop.

Software changes cannot repair an intermittent cable. Repeated disconnect and
reconnect messages in the kernel journal are strong evidence of a physical,
power, or link problem.

### Observe enumeration live

Run one observer, then connect the device:

```bash
sudo journalctl -kf
```

or:

```bash
sudo udevadm monitor --kernel --udev --property
```

In another terminal:

```bash
lsusb
lsusb -t
```

Record vendor ID, product ID, product name, serial where present, USB bus
topology, kernel driver, and assigned subsystem. Do not paste an entire
unbounded journal into an issue; capture the connection window and relevant
identifiers.

### Device nodes are not stable identities

Names such as these are allocation results:

```text
/dev/ttyUSB0
/dev/ttyACM0
/dev/video0
/dev/lp0
/dev/sdb
```

They can change after reboot, reconnect, a different port, or the addition of
another device. Prefer stable identities exposed by udev when available:

```bash
ls -l /dev/serial/by-id/ /dev/serial/by-path/ 2>/dev/null
ls -l /dev/disk/by-id/ 2>/dev/null
readlink -f /dev/serial/by-id/EXACT_DEVICE_NAME
```

Use `by-id` for a device with a useful unique serial and `by-path` when the
physical port is intentionally part of identity. Do not put `/dev/ttyUSB0` or
`/dev/lp0` into a long-lived script just because it worked once.

Inspect udev properties for an exact current node:

```bash
udevadm info --query=all --name=/dev/DEVICE
```

The placeholder must be replaced with the observed node. Properties such as
`ID_VENDOR_ID`, `ID_MODEL_ID`, `ID_SERIAL`, `TAGS`, and `CURRENT_TAGS` help
explain identity and whether seat-based access was requested; they do not by
themselves prove that the current process has access.

### Permissions: inspect owner, group, ACL, and seat

Modern desktop access can come from a device owner/group, a packaged udev
rule, or a `uaccess` ACL assigned to the active local seat. Inspect all of
them:

```bash
stat /dev/DEVICE
getfacl /dev/DEVICE
loginctl session-status
id
```

For a serial device on Arch, the node may be owned by a group such as `uucp`.
That is evidence to evaluate, not permission to add the user pre-emptively.
If group membership is genuinely the packaged access model, add only the
required group and start a fresh login session afterward:

```bash
sudo usermod -aG uucp neon
```

This command changes account membership and is conditional. Confirm the exact
node ownership first. Never use `chmod 666` on device nodes as a persistent
repair: udev will recreate the node, the change is overbroad, and it conceals
the missing policy. Do not run an IDE, slicer, scanner GUI, or browser as root
to bypass access controls.

A local udev rule is the last narrow option when the device has stable
vendor/product/serial identifiers and no package supplies a correct rule. The
rule must match only the intended hardware, document its access rationale,
and be validated before reload. A rule matching an entire USB class or every
device from one vendor is too broad.

### One process may own an interface exclusively

A correct permission error can actually be contention. Inspect an exact node:

```bash
lsof /dev/DEVICE
fuser -v /dev/DEVICE
```

Serial monitors, IDEs, vendor tools, camera applications, classic USB printer
backends, and `ipp-usb` can each hold interfaces. Close or reconfigure the
identified owner; do not disable an unrelated daemon based on folklore.

### Power management is a separate experiment

If a device works after boot or on AC power but disappears on battery, after
idle, or after resume, inspect power policy:

```bash
sudo tlp-stat -u
journalctl -b -k --grep='usb\|xhci\|disconnect\|reset' --no-pager
journalctl -b -u tlp.service --no-pager
```

Compare one variable at a time. A per-device TLP exclusion is preferable to
globally disabling USB autosuspend, but only after the journal and a controlled
test identify power management as the cause. Guide 13 owns TLP and suspend
policy.

### Input, cameras, and other classes

Use the subsystem's own tools after USB enumeration:

| Device role | Useful inspection | Project boundary |
| --- | --- | --- |
| Keyboard, mouse, tablet | `libinput list-devices`, `libinput debug-events` | Do not change `/dev/input` permissions broadly |
| Webcam/UVC camera | `v4l2-ctl --list-devices` when `v4l-utils` is installed | Sandboxed/web access may also involve PipeWire portals |
| Serial adapter or board | `/dev/serial/by-id`, `udevadm info`, `lsof` | Use stable identity and narrow access |
| USB storage | `lsblk`, UDisks, GVfs | Guide 12 owns mounting and safe removal |
| Android/MTP device | `gio mount -l`, GVfs MTP | Device must expose the intended USB mode |
| Bluetooth peripheral | `bluetoothctl`, BlueZ journal | Guide 12 owns pairing and trust |

A camera can expose storage/MTP and UVC video as different interfaces. A
multifunction printer can expose IPP-over-USB, eSCL, storage, and vendor
interfaces simultaneously. Diagnose the intended function, not only the
product name.

## Layered diagnosis

### Printer not shown anywhere

1. Confirm power, cable, network association, and device mode.
2. For USB, inspect `journalctl -kf`, `lsusb`, and `lsusb -t`.
3. For network, verify the exact address and route:

   ```bash
   ip route get PRINTER_ADDRESS
   ping -c 3 PRINTER_ADDRESS
   ```

4. Query the device's documented IPP endpoint with the CUPS tools rather than
   opening firewall services blindly.
5. If using discovery, perform one bounded mDNS/WSD test on a trusted LAN.

Ping can be blocked while IPP works, so a failed echo response is not final
proof. It only locates the next test.

### Printer discovered but queue creation fails

Inspect:

```bash
systemctl status cups.socket cups.service
sudo cupsd -t
lpinfo -v
journalctl -b -u cups.service --no-pager
journalctl -b _COMM=polkitd --no-pager
```

Separate an unreachable device URI from denied administration. A polkit prompt
failure does not imply that the printer driver is wrong.

### Queue exists but jobs remain pending

Inspect:

```bash
lpstat -t
lpstat -W all -o QUEUE_NAME
lpoptions -p QUEUE_NAME -l
journalctl -u cups.service --since '-15 min' --no-pager
```

Then inspect the device panel, paper, supplies, network address, and sleep
state. Do not recreate the queue before recording its URI and options.

### Filter or format failure

Confirm the queue uses `everywhere` when the printer supports it. Check the
exact job and CUPS journal for a named filter. If a legacy driver is involved,
verify that its package still owns the filter:

```bash
pacman -Qo /usr/lib/cups/filter/EXACT_FILTER
pacman -Qkk PACKAGE_NAME
```

Do not install several driver suites until one happens to silence the error.
That makes the effective conversion path harder to prove.

### Scanner appears in USB but not in SANE

```bash
lsusb
sane-find-scanner
scanimage -L
SANE_DEBUG_DLL=3 scanimage -L
```

If low-level tools see hardware but `scanimage -L` does not, inspect backend
support, backend configuration, and device access. If the multifunction device
uses IPP-over-USB, also inspect `ipp-usb` before adding a classic backend.

### Network scanner not discovered

1. Verify its known eSCL URL if available.
2. Inspect the active NetworkManager connection and firewalld zone.
3. Run `airscan-discover` during a bounded discovery window.
4. Compare DNS-SD and WSD behavior.
5. Prefer a reviewed manual `[devices]` entry if discovery is unreliable.

Do not enable `saned.socket`; that serves local scanners to other computers
and does not fix discovery of a remote eSCL device.

### Device works only as root

That observation proves an access difference, not the correct repair.
Inspect:

```bash
stat /dev/DEVICE
getfacl /dev/DEVICE
udevadm info --query=all --name=/dev/DEVICE
id
loginctl session-status
```

Then determine whether a packaged group, `uaccess` tag, polkit action, or
application sandbox owns the boundary. Stop running the application as root
after collecting the evidence.

### Device disappears after resume

Capture both the suspend and resume window:

```bash
journalctl -b --since '10 minutes ago' --no-pager
journalctl -b -k --grep='usb\|xhci\|reset\|disconnect' --no-pager
sudo tlp-stat -u
```

Test direct connection without a dock and compare AC/battery behavior. Change
one device-specific power variable only after the failure is reproducible.

## Logs and temporary debug state

Normal systemd logs come first:

```bash
journalctl -b -u cups.service --no-pager
journalctl -b -u ipp-usb.service --no-pager
journalctl -b -k --no-pager
```

If ordinary CUPS logs are insufficient, enable debug logging for a bounded
reproduction:

```bash
sudo cupsctl --debug-logging
```

Reproduce one job, collect the relevant interval, and disable it immediately:

```bash
sudo cupsctl --no-debug-logging
```

Debug traces can contain printer addresses, usernames, document titles, job
metadata, and protocol data. Review and redact them before publication.
`printers.conf` can contain credentials embedded in device URIs and is
deliberately protected by CUPS; do not paste it wholesale into an issue or
commit it to Git.

## Safe rollback and recovery

### Remove one mistaken queue

Record its state first:

```bash
lpstat -p QUEUE_NAME -l
lpstat -v QUEUE_NAME
lpoptions -p QUEUE_NAME -l
```

Deleting a queue aborts its active and pending jobs. When the exact destination
has been confirmed:

```bash
sudo lpadmin -x QUEUE_NAME
```

This does not uninstall CUPS or alter the physical printer.

### Revert temporary discovery

A firewalld `--timeout` rule expires automatically. Inspect current runtime
state before removing anything:

```bash
sudo firewall-cmd --zone=public --list-services
```

If the exact runtime `mdns` rule still exists and should end immediately:

```bash
sudo firewall-cmd --zone=public --remove-service=mdns
```

Do not reload firewalld if unrelated runtime work must be preserved. Do not
edit nftables rules directly while firewalld owns policy.

### Restore local-only CUPS policy

Validate configuration and reassert only the selected controls:

```bash
sudo cupsd -t
sudo cupsctl --no-share-printers --no-remote-admin --no-remote-any
cupsctl
ss -lntup
```

If manual configuration edits caused the failure, restore the exact backup or
merge the intended directives rather than replacing `/etc/cups` wholesale.
Use `pacdiff` for package update files; `printers.conf` is CUPS-managed state
and should not be hand-edited.

### Back out a conflicting USB path

If installing `ipp-usb` made a previously working classic USB driver lose the
device, stop and inspect which interface now owns it. Choose one path, disable
or remove only the incompatible selected component, reconnect the device, and
repeat `lpinfo -v` plus `scanimage -L`. Do not alternate services rapidly
without recording the result of each state.

### Preserve a recovery path

Before changing printing or peripheral access:

- keep a TTY available;
- record installed packages and enabled units;
- save exact queue URIs and defaults without publishing credentials;
- back up local files before editing `/etc/cups`, `/etc/sane.d`, or udev rules;
- keep firewall tests runtime-only until proven;
- know how to remove the exact queue, rule, group membership, or local file.

## Security and privacy boundaries

Printing and scanning handle real document contents. Treat them as data paths,
not merely hardware conveniences.

- Prefer IPPS when the printer supports and correctly identifies it.
- Do not accept an unexpected printer certificate or downgrade silently
  without understanding the local threat model.
- Keep printer administration and sharing on loopback/local policy unless a
  server role is deliberately designed.
- Avoid publishing device names, addresses, serial numbers, job titles, or
  document paths in public logs.
- Review proprietary filters and vendor daemons as executable code with system
  access.
- Never embed printer credentials in dotfiles or public documentation.
- Do not grant whole USB classes or device subsystems world-writable access.
- Keep Avahi, `cups-browsed`, `saned`, and inbound firewall services disabled
  when their discovery or server role is not needed.

## Decision checklist for a new device

Record these answers before changing the system:

1. What is the exact manufacturer, model, hardware revision, and firmware?
2. Is the required function printing, scanning, storage, camera, serial, HID,
   or several independent interfaces?
3. Does it support IPP Everywhere, IPPS, IPP-over-USB, eSCL, or WSD?
4. Can it use a known stable address instead of automatic discovery?
5. Which package owns the necessary backend, rule, filter, or daemon?
6. Which system and user units would be activated?
7. Does the laptop act only as a client, or would the change create a server?
8. Which firewalld zone and network trust boundary apply?
9. Is access based on group ownership, `uaccess`, polkit, or an application
   sandbox?
10. How will enumeration, protocol function, application function, and resume
    behavior be tested separately?
11. What is the exact rollback for the queue, service, rule, group membership,
    or configuration file?

## Project decisions

The resulting policy is:

- printing and scanning remain conditional features;
- IPP Everywhere and eSCL/AirScan are preferred over model-specific drivers;
- a known URI is preferred over automatic multicast discovery;
- `cups.socket` provides on-demand local scheduler activation;
- every configured queue is explicitly unshared;
- CUPS remote administration and remote-any access remain disabled;
- Avahi and `cups-browsed` remain off until a real discovery workflow exists;
- `saned.socket` remains disabled because the laptop is not a scanner server;
- `ipp-usb` is selected only for a compatible device and not stacked blindly
  with a classic USB driver;
- `cups-pk-helper` uses polkit without a blanket project rule;
- group membership and local udev rules require device-specific evidence;
- stable udev identities replace volatile device-node numbers in scripts;
- TLP exceptions are per-device and evidence-based;
- no inbound print, discovery, or scanner firewalld service is permanent by
  default.

## Sources and further reading

- [ArchWiki: CUPS](https://wiki.archlinux.org/title/CUPS)
- [ArchWiki: SANE](https://wiki.archlinux.org/title/SANE)
- [ArchWiki: udev](https://wiki.archlinux.org/title/Udev)
- [OpenPrinting: command-line printer administration](https://openprinting.github.io/cups/doc/admin.html)
- [OpenPrinting: printer sharing](https://openprinting.github.io/cups/doc/sharing.html)
- [`lpadmin(8)`](https://openprinting.github.io/cups/doc/man-lpadmin.html)
- [`cupsctl(8)`](https://openprinting.github.io/cups/doc/man-cupsctl.html)
- [`cups-files.conf(5)`](https://openprinting.github.io/cups/doc/man-cups-files.conf.html)
- [`sane-airscan(5)`](https://man.archlinux.org/man/sane-airscan.5.en)
- [`airscan-discover(1)`](https://man.archlinux.org/man/airscan-discover.1.en)
- [`scanimage(1)`](https://man.archlinux.org/man/scanimage.1.en)
- [OpenPrinting `ipp-usb`](https://github.com/OpenPrinting/ipp-usb)
- [`firewall-cmd(1)`](https://firewalld.org/documentation/man-pages/firewall-cmd.html)

The next handbook extension is guide 24: Plymouth, early-boot presentation,
encrypted-root prompts, UKI integration, and recovery.
