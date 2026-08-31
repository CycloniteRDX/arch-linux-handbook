# Niri outputs, scaling, external displays, and host overrides

## Purpose and scope

A monitor configuration is more than a resolution. The compositor must decide
which physical displays are active, which advertised timing each one uses, how
large graphical objects should appear, where each logical desktop rectangle
sits, which monitor receives focus, and what happens when hardware disappears.

On a laptop, that topology also meets a separate power policy:

- Niri controls the internal and external output state;
- the kernel reports connectors, modes, EDID metadata, and lid switches;
- systemd-logind decides whether a lid event suspends the machine;
- swayidle can power displays down temporarily without disabling them;
- WirePlumber independently chooses an HDMI or DisplayPort audio route;
- applications and portals consume the resulting logical layout.

This guide explains those boundaries and records a host-override design for
the two ThinkPad T14 Gen 1 AMD installations. It does not yet activate a fixed
mode or scale on either machine: only one internal panel has partial measured
evidence, the second panel still needs a complete record, and no recurring
external-display topology has been accepted.

The portable Niri configuration therefore remains safe while the measurements
are completed.

## Current project evidence

The two canonical computers are Lenovo ThinkPad T14 Gen 1 AMD machines, one
with a Ryzen 5 PRO 4650U and one with a Ryzen 7 PRO 4750U. Hardware generation
does not guarantee identical panels: Lenovo configurations, replacement
screens, firmware tables, and exact timings can differ.

The evidence currently recorded is:

| Item | Known state |
| --- | --- |
| Portable Niri file | Contains no active `output` block |
| Internal connector seen on one T14 | `eDP-1` |
| Reported native resolution | `1920x1080` |
| Reported refresh timings | Approximately `60.049` Hz and `48.040` Hz |
| Scale experiment | `1.5` was useful enough to consider, not yet canonical |
| Second T14 panel | Exact identity, modes, scale, and physical preference not yet recorded here |
| External monitor topology | Still conditional and hardware-dependent |
| Current default | Let Niri choose preferred mode, guessed scale, and automatic placement |

The opening comment in the tracked configuration states this contract:

```kdl
// Portable daily-driver configuration for the two ThinkPad installations.
// Output modes and scale remain automatic until measured on each machine.
```

This is not missing configuration. Niri automatically enables connected
outputs, chooses a suitable mode and scale, and places outputs when no explicit
rule overrides that behavior. Automatic operation is the correct baseline
while evidence is incomplete.

## The output model

### Display, output, connector, and panel

These terms describe different layers:

| Term | Meaning |
| --- | --- |
| Physical panel or monitor | The actual LCD/OLED device and its electronics |
| GPU connector | The DRM/KMS connection exposed by the kernel, such as `eDP-1`, `HDMI-A-1`, or `DP-2` |
| Output | The compositor's controllable display endpoint associated with a connector |
| Mode | A resolution plus timing/refresh combination advertised or configured for that output |
| EDID | Monitor-supplied metadata that may contain manufacturer, model, serial, physical size, and modes |
| Logical output rectangle | The compositor-space width, height, and position after scale and transform |

The built-in laptop panel is usually exposed through embedded DisplayPort and
named `eDP-1`. That name describes the connector relationship, not the panel
model. Replacing the LCD can leave the connector name unchanged while changing
the EDID and supported timings.

External connector names are less stable. A USB-C dock, GPU probe order, or
different physical port can make the same monitor appear through another
`DP-*` name. Conversely, two identical monitors can report the same make and
model and may even expose missing or duplicated serial fields.

There is no universally best identifier. Identity is chosen from measured
behavior and the scope of the rule.

### Niri output matching

Niri accepts either:

- a connector name, such as `eDP-1`; or
- the exact manufacturer, model, and serial identity reported by
  `niri msg outputs`.

Example forms are:

```kdl
output "eDP-1" {
    // Built-in connector on this host profile.
}

output "Some Manufacturer Some Model Some Serial" {
    // Physical-display identity when all fields are reliable and suitable.
}
```

Use connector matching when the port is the intended identity. Use monitor
metadata when the physical display should retain policy across ports and the
reported identity is stable and unique.

Do not copy a monitor serial number into a public repository. A raw EDID record
or `niri msg outputs` capture can expose a persistent hardware identifier. The
host design selected later in this guide deliberately needs no serial number
for the built-in panels.

## Observe before configuring

Run these commands inside the active Niri session:

```bash
niri msg outputs
niri msg --json outputs | jq .
niri msg workspaces
loginctl session-status
```

For every output, record:

- connector name;
- manufacturer, model, and serial, with the serial kept private;
- physical dimensions if reported;
- current and preferred mode;
- every available resolution and exact refresh timing;
- current scale;
- transform;
- logical position and logical dimensions;
- whether variable refresh rate is supported;
- connection path: internal, HDMI, USB-C, or dock;
- whether the result changes after unplugging, rebooting, or resuming.

Keep the unredacted machine record below the backed-up System Records
directory, not in the public handbook or dotfiles repository:

```bash
mkdir -p ~/Documents/System-Records
niri msg --json outputs | jq . | tee ~/Documents/System-Records/niri-outputs-$(date +%F).json
```

The filename date distinguishes measurements taken before and after a kernel,
firmware, dock, cable, or panel change. Restic already protects this directory
under the project's backup policy.

The human-readable output is best for immediate inspection. JSON is better for
comparison or scripts because columns and prose can change between releases.
Consumers should tolerate added JSON fields.

### A measurement worksheet for each ThinkPad

Use one private record per computer:

```text
Machine label:
Hardware model:
Niri version:
Kernel version:
Connector:
Manufacturer/model:
Serial retained privately: yes/no
Physical size:
Preferred mode:
Other supported modes:
Automatic scale:
Scale candidates tested:
Chosen scale and reason:
Logical dimensions:
AC behavior:
Battery behavior:
Lid without external display:
Lid with external display:
External connector path:
Hot-plug result:
Suspend/resume result:
Notes and rollback:
```

Do not fill an unknown field with a guessed value. `NOT TESTED` and
`NOT REPORTED` distinguish missing evidence from a successful test.

## Modes and refresh rates

### A mode is an advertised timing

The string:

```text
1920x1080@60.049
```

means a 1920 by 1080 pixel mode whose reported refresh timing is 60.049 Hz.
The decimal value is not cosmetic. If a Niri output block includes a refresh
rate, it must match the value reported by `niri msg outputs`, including the
three decimal digits.

Valid configuration forms include:

```kdl
output "eDP-1" {
    mode "1920x1080@60.049"
}
```

or:

```kdl
output "eDP-1" {
    mode "1920x1080"
}
```

Omitting the refresh asks Niri to choose the highest advertised refresh for
that resolution. Omitting `mode` entirely lets Niri select automatically. If
the preferred automatic choice is already the desired 60.049 Hz mode, writing
the same mode explicitly adds maintenance cost without changing behavior.

Do not round `60.049` to `60`, copy a timing from the other T14, or invent a
modeline. Both 60.049 Hz and 48.040 Hz are already advertised on the measured
panel, so a custom mode is unnecessary.

### 60 Hz versus the reported 48 Hz mode

The likely trade-off on the measured internal panel is:

| Mode | Expected advantage | Expected cost |
| --- | --- | --- |
| `1920x1080@60.049` | Smoother pointer, scrolling, animation, and video cadence | Potentially somewhat higher panel/display-engine power |
| `1920x1080@48.040` | Possible battery saving | Less fluid interaction and awkward cadence for some content |

The energy benefit must be measured rather than assumed. Panel power,
backlight brightness, workload, compositor redraw, Wi-Fi, CPU package state,
and battery condition can dominate a small refresh-rate difference.

Keep brightness and workload constant during any comparison. Observe battery
energy rate over a long enough interval for noise to settle, then repeat the
order on another run. One instantaneous `upower` value is not evidence of a
real saving.

The current project keeps 60 Hz or the preferred automatic mode as the normal
interactive baseline. A 48 Hz battery policy remains an accepted future
experiment, not part of TLP and not an implicit consequence of selecting
`power-saver`.

TLP controls system power policy. Niri controls output timing. Connecting them
requires an explicit, tested integration owner; neither program should poll
and override the other through an improvised loop.

## Physical pixels, logical pixels, and scale

### Physical density

For an unrotated panel, approximate pixel density is:

```text
PPI = sqrt(width_px² + height_px²) / diagonal_inches
```

A 14-inch 1920×1080 panel is roughly 157 PPI. This explains why scale 1.0 can
make text and controls feel small at normal laptop distance, but PPI alone does
not select the correct scale. Eyesight, viewing distance, application mix, font
choices, and desired workspace all matter.

### Logical size

At a simple scale factor:

```text
logical width  = physical width  / scale
logical height = physical height / scale
```

For the 1920×1080 internal mode:

| Scale | Approximate logical rectangle | Practical effect |
| --- | --- | --- |
| `1.0` | 1920×1080 | Maximum workspace, smallest UI |
| `1.25` | 1536×864 | Moderate enlargement and still substantial workspace |
| `1.5` | 1280×720 | Larger UI, less room for columns and vertical content |
| `2.0` | 960×540 | Usually too little logical workspace for this 14-inch FHD use case |

The hardware continues scanning 1920×1080 pixels at scale 1.5. Scale changes
the logical coordinate system and the size at which clients render; it does
not reduce the panel mode to 1280×720.

Niri can guess scale from resolution and physical dimensions when `scale` is
omitted. EDID physical dimensions can be absent or inaccurate, so the guessed
value is a starting point rather than a personal ergonomic verdict.

### Fractional scaling

Niri accepts fractional values such as:

```kdl
output "eDP-1" {
    scale 1.5
}
```

Fractional scaling is normal in modern Wayland compositors, but it introduces
rounding between logical and physical grids. Niri performs its layout with
fractional logical coordinates and aligns relevant geometry to physical pixels
to avoid uneven borders, gaps, and blur.

Applications still differ:

- native Wayland clients can react to output scale and render a suitable
  buffer;
- a window moved between differently scaled outputs may need to rerender;
- older applications can use integer assumptions;
- X11 applications running through Xwayland may have less coherent mixed-DPI
  behavior than native Wayland clients.

Test the actual application set: Firefox, Kitty, Nautilus, Papers, Loupe,
LibreOffice, Code - OSS if installed, and at least one known X11 client. Look
for readable text, crisp icons, usable dialogs, correct pointer mapping, and
reasonable window sizes on every output.

Do not set global `GDK_SCALE`, `QT_SCALE_FACTOR`, or toolkit-backend variables
to repair one application. Global overrides can double-scale native clients,
break portals, and make the same configuration wrong on another monitor.

### Scale and Niri's proportional layout

The portable Niri layout expresses column widths as proportions:

```kdl
preset-column-widths {
    proportion 0.33333
    proportion 0.5
    proportion 0.66667
}
```

Those proportions adapt to each output's logical width. A half-width column on
a 1920-logical-pixel output is much wider in logical pixels than a half-width
column on a 1280-logical-pixel output, but both occupy half the workspace.

This is why the shared layout can remain portable while output scale is
host-specific. Fixed logical widths would need more per-output tuning.

## Logical coordinates and monitor placement

### Position uses scaled dimensions

Niri places outputs in a global logical coordinate space. Mode resolution,
scale, and rotation determine each rectangle's logical size before `position`
is evaluated.

For example, a 3840×2160 monitor at scale 2 has a 1920×1080 logical rectangle.
To place another output immediately to its right, that second output begins at
`x=1920`, not `x=3840`.

An illustrative configuration is:

```kdl
output "External Manufacturer Model Serial" {
    mode "3840x2160@60.000"
    scale 2
    position x=0 y=0
}

output "eDP-1" {
    mode "1920x1080@60.049"
    scale 1.5
    position x=1920 y=180
}
```

This example is arithmetic, not a profile to copy. The external identity,
timing, physical arrangement, and desired vertical alignment have not been
measured for this project.

### Adjacency matters

The pointer can cross only where logical output edges are directly adjacent.
If a small laptop rectangle is vertically centered beside a tall monitor,
only the overlapping edge segment connects them. A pointer pushed against a
non-overlapping part remains on the current output.

Choose positions from the physical desk arrangement:

- left versus right;
- aligned top edges, bottom edges, or vertical centers;
- portrait transform;
- laptop below an external monitor;
- lid-open dual display versus lid-closed external-only use.

Do not invent coordinates from physical resolution alone. Compute logical
dimensions after the final scale and transform, then draw the rectangles on
paper if the topology is not obvious.

### Automatic placement

When positions are absent, Niri sorts connected outputs by name and places
them from left to right. Explicit positions are applied first; overlapping or
invalid placements are moved automatically and generate a warning.

Automatic placement is sufficient for occasional projectors, televisions, or
an external monitor whose physical arrangement changes. Static positions are
appropriate only for a repeated desk layout.

## Output settings that are deliberately not enabled

An output block can also control:

| Option | Meaning | Current policy |
| --- | --- | --- |
| `off` | Remove the output from the active topology | Not set for either built-in panel |
| `transform` | Rotate or flip logical presentation | Automatic normal orientation |
| `variable-refresh-rate` | Enable adaptive refresh when supported | Not enabled without a proven use case and test |
| `focus-at-startup` | Prefer an output for initial focus | Automatic until topology is stable |
| `max-bpc` | Limit bits per color channel | Leave driver/display default |
| Custom mode/modeline | Request a non-advertised timing | Rejected for the reported laptop modes |
| Per-output layout | Override gaps or layout on one monitor | Shared proportional layout remains adequate |

VRR can help some games, but drivers and monitors can exhibit cursor stutter,
black flashes, or repeated modesets. It is not a generic quality switch. If a
future external gaming monitor supports it, test `on-demand=true` with an
explicit application rule before considering always-on VRR.

Color calibration, HDR, wide gamut, and ICC profile handling are separate
projects. Raising color depth or adding a profile without a measured display
pipeline is not calibration.

## Temporary monitor power is not `off`

The current swayidle policy uses:

```bash
niri msg action power-off-monitors
niri msg action power-on-monitors
```

This is temporary display power management. The outputs remain part of the
session topology, and activity can restore them. It is analogous to letting
the screens sleep while applications and workspaces continue to exist.

By contrast:

```kdl
output "eDP-1" {
    off
}
```

is persistent configuration that removes the output until the rule changes.
It should not be used to implement the ten-minute idle timeout.

The future third idle stage will suspend the computer after locking and
powering off the monitors. It will still not rewrite output configuration or
log out of Niri.

## Lid behavior crosses two owners

### Niri's part

Niri automatically turns the internal laptop output off when the lid closes
and on when it opens. A custom `switch-events` binding is not required for that
basic behavior.

Do not add a lid-close script that also disables `eDP-1` unless an observed bug
requires it. The duplicate owner can race Niri's native behavior and remain
wrong after resume.

### systemd-logind's part

The existing local logind policy is:

```ini
HandleLidSwitch=suspend
HandleLidSwitchExternalPower=suspend
HandleLidSwitchDocked=ignore
```

Its intended outcomes are:

| Context | logind result | Niri output result |
| --- | --- | --- |
| Laptop alone on battery | Suspend | Internal output turns off as lid closes |
| Laptop alone on AC | Suspend | Internal output turns off as lid closes |
| Docked or more than one display connected | Continue running | Internal output turns off; external output remains |

The third case enables lid-closed use with an external monitor. logind treats
more than one connected display as docked for lid-policy selection; a branded
docking station is not required.

Inspect the effective policy rather than only its local drop-in:

```bash
systemd-analyze cat-config systemd/logind.conf
loginctl seat-status seat0
```

Niri deciding that `eDP-1` is off does not decide whether the machine sleeps.
logind ignoring a docked lid does not decide the external monitor's mode or
scale. Both results must be tested together.

### Required lid tests

With all work saved:

1. Disconnect external displays and close the lid on battery.
2. Confirm suspend and resume behind swaylock.
3. Repeat on AC.
4. Connect an external display and prove both outputs are active.
5. Close the lid and confirm the machine remains running on the external
   display.
6. Confirm the internal output disappears from the active topology.
7. Reopen the lid and confirm the internal output returns.
8. Suspend manually with both displays and verify recovery.
9. Disconnect the external display and confirm all work returns to the
   internal panel.

Inspect the same boot afterward:

```bash
journalctl -b --no-pager | grep -Ei 'lid|suspend|resume|sleep|drm|amdgpu|niri'
```

If no external monitor is available, record the docked cases as `NOT TESTED`.

## Hot-plug and Niri workspaces

Niri gives every output its own vertical sequence of dynamic workspaces. The
model is not “one global workspace number shown on a chosen monitor”; each
monitor owns its current workspace set.

When an output disconnects, Niri moves its workspaces to a remaining output.
It remembers their origin and can move them back when the monitor reconnects.
This is a major reason not to kill applications or restart Niri during ordinary
hot-plug.

Observe the state before and after connecting hardware:

```bash
niri msg outputs
niri msg workspaces
```

Useful manual IPC tests are:

```bash
niri msg action focus-monitor-right
niri msg action focus-monitor-left
niri msg action move-workspace-to-monitor-right
niri msg action move-workspace-to-monitor-left
```

Run only an action that makes sense for the observed topology. A direction
with no adjacent output can do nothing.

The current portable binding set focuses and moves columns inside a monitor
but does not yet record a complete family of monitor-direction shortcuts.
After both machines pass external-display tests, those bindings can be added
as a separate portable dotfiles decision.

Test this lifecycle:

1. open harmless windows on both outputs;
2. create more than one workspace on the external display;
3. disconnect the cable normally;
4. verify every window and workspace remains reachable internally;
5. reconnect through the same port;
6. verify the external workspaces return where possible;
7. repeat after suspend and resume;
8. repeat once through any intended USB-C dock.

Do not terminate “duplicate” workspaces based only on their count. Inspect
their output association first.

## Video and audio are separate paths

HDMI and DisplayPort can expose both a display and an audio sink, but Niri owns
only the output topology. PipeWire and WirePlumber own audio routing.

After connecting an external display:

```bash
niri msg outputs
wpctl status
```

A correct picture does not prove external audio is selected. A correct audio
sink does not prove scale, refresh, or logical position. Diagnose each owner
with the audio guide's tools.

The same separation applies to brightness. `brightnessctl` normally controls
the laptop backlight through the kernel. Many external monitors require their
own on-screen controls or DDC/CI; Niri scale and mode do not control panel
brightness.

## Selected host-override architecture

### Goals

The design must:

- retain one shared portable `config.kdl`;
- allow the two internal panels to use different exact timings or scales;
- avoid runtime scripts that guess the machine from `$HOSTNAME`;
- keep host selection explicit and reversible;
- use GNU Stow consistently;
- avoid committing monitor serial numbers or secrets;
- leave a clean clone usable before a host profile is selected;
- prevent both host profiles from being active simultaneously.

### Portable base plus one optional include

Once both machines are measured and the implementation commit is prepared,
the shared `config.kdl` will end with:

```kdl
include optional=true "~/.config/niri/host.kdl"
```

Niri 26.04 supports both `optional=true` and `~` expansion. The absolute
home-relative path matters because the tracked `config.kdl` is a Stow symlink;
the local override must be found beside the deployed configuration, not by
following assumptions about the repository target path.

An optional include lets a fresh clone start with automatic output behavior
before a host package is selected. Niri emits a warning while the optional
file is missing, which makes the incomplete selection visible without making
the configuration invalid.

The include belongs at the end because includes are positional: later values
can override earlier mergeable settings. Output sections themselves are
multipart entries inserted as complete rules rather than merged field by
field. The portable base must therefore continue to omit an `output "eDP-1"`
block; duplicating the same output in the base and host file is invalid or
ambiguous.

### Two explicit Stow packages

The future repository layout will be:

```text
hosts/
├── t14-r5/
│   └── .config/
│       └── niri/
│           └── host.kdl
└── t14-r7/
    └── .config/
        └── niri/
            └── host.kdl
```

The directory labels identify hardware profiles, not network hostnames. Each
file begins with a comment recording which measured machine record justifies
it, but no chassis serial, panel serial, username, or secret.

Only one package is deployed on a machine. For the Ryzen 7 profile, the future
commands will be:

```bash
stow --dir=hosts --simulate --verbose --no-folding --target="$HOME" t14-r7
stow --dir=hosts --verbose --no-folding --target="$HOME" t14-r7
niri validate
```

The Ryzen 5 machine selects `t14-r5` instead. Stow stops on a conflict if
another package already owns `~/.config/niri/host.kdl`; that is a safety
feature.

To switch deliberately:

```bash
stow --dir=hosts --delete --verbose --target="$HOME" t14-r7
stow --dir=hosts --simulate --verbose --no-folding --target="$HOME" t14-r5
stow --dir=hosts --verbose --no-folding --target="$HOME" t14-r5
niri validate
```

These commands are architecture examples, not instructions to run now. The
`hosts/` tree and include will be added only after the second ThinkPad record
and chosen scale values exist.

### Scope of each `host.kdl`

Initially, a host file may contain only measured internal-output policy:

```kdl
// Example shape only; copy exact evidence for this machine.
output "eDP-1" {
    mode "1920x1080@60.049"
    scale 1.5
}
```

The first real profile may omit `mode` if automatic preferred mode is already
correct and only scale differs. Minimal host files are easier to audit and
survive panel replacement.

Do not add keyboard layout, TrackPoint tuning, wallpapers, power scripts,
external monitor serials, or application state merely because the file is
host-specific. Those concerns have their own owners and review criteria.

If a future fixed desk monitor genuinely needs serial-based matching, keep the
raw identifier in a private machine record. Decide separately whether a
sanitized connector/model rule is stable enough for the public dotfiles or a
small local-only fragment must remain Restic-protected.

## Why Kanshi is not installed now

Static Niri output rules answer:

> When this output exists, what policy should it use?

They do not naturally express:

> Use one complete arrangement when only the laptop is connected, another
> when monitor A appears, and a third when the dock exposes A and B together.

Niri's upstream FAQ recommends Kanshi for connection-dependent profiles.
Kanshi watches outputs and applies a matching profile containing enablement,
mode, scale, position, transform, and optional commands.

That capability is unnecessary until a repeated topology exists. Today:

- both laptops work with automatic output selection;
- only one internal panel is partially recorded;
- no accepted dock/desk layout requires conditional profiles;
- the lid behavior already has native Niri and logind owners.

Installing Kanshi now would add a second configuration language and another
long-running session owner without solving a measured problem.

If it is added later, ownership must be explicit:

| Concern | Owner in a future Kanshi design |
| --- | --- |
| Stable per-output defaults | Either Niri host file or Kanshi defaults, not contradictory values in both |
| Connected-set profile matching | Kanshi |
| Output application | Kanshi through the compositor's output-management interface |
| Lid suspend decision | systemd-logind |
| Internal panel lid on/off | Niri native behavior |
| Idle monitor power | swayidle plus Niri action |

Kanshi should be introduced with a clean autostart owner, profile validation,
hot-plug tests, and a documented way to stop it. It must not race static Niri
rules that continually request different values.

## Safe change procedure

### Before adding any output rule

1. Save the raw output report privately.
2. Confirm the exact machine and Niri version.
3. Keep TTY3 logged in.
4. Save all graphical work.
5. Change only one property first: scale, mode, or position.
6. Validate before relying on live reload.
7. Confirm the display remains usable.
8. Test logout/login and then one reboot.

Commands:

```bash
hostnamectl
pacman -Q niri
niri msg outputs
niri validate --config ~/.config/niri/config.kdl
```

Niri live-reloads valid output settings, but a black screen is still possible
if an output is explicitly turned off or a topology becomes unusable. Live
reload reduces iteration time; it does not replace a recovery route.

### Scale test

For each candidate scale:

1. inspect logical dimensions;
2. open all daily applications;
3. test dialogs, menus, tooltips, and file pickers;
4. move native Wayland and XWayland windows between outputs;
5. inspect Waybar, Fuzzel, Mako, swaylock, and screenshots;
6. work long enough to assess readability and usable space;
7. record the result rather than choosing from one screenshot.

Scale is an ergonomic preference backed by technical validation. There is no
score that makes 1.25 objectively superior to 1.5 for every user.

### Mode test

For each advertised refresh mode:

1. verify the exact string;
2. save work before the modeset;
3. check for blanking, flicker, artifacts, or unstable resume;
4. evaluate pointer, scrolling, animation, and video;
5. perform several suspend/resume cycles;
6. measure battery effect separately if that is the goal.

Do not test a custom mode when the desired timings are already advertised.

### External-display test

Record each state:

```bash
niri msg outputs
niri msg workspaces
wpctl status
journalctl --user -b -u niri.service --no-pager
```

Test:

- hot-plug and unplug;
- cursor crossing along the intended edge;
- focus and workspace movement;
- mixed scale and application rerendering;
- lid-open and lid-closed operation;
- external audio selection;
- monitor power-off and wake;
- suspend/resume;
- logout/login;
- one cold boot with the monitor already connected.

Cable bandwidth matters. A mode that appears through direct USB-C or HDMI may
be absent through a dock, adapter, or lower-spec cable. Do not force the direct
connection's timing onto a path that does not advertise it.

## Diagnosis by symptom

| Symptom | Likely boundary | First checks |
| --- | --- | --- |
| Internal panel has the wrong size | Scale guess or host override | `niri msg outputs`; active `host.kdl` link |
| Requested mode is ignored | Inexact refresh string or mode unavailable on this path | Current mode list; cable/dock path; Niri journal |
| Pointer cannot cross at part of an edge | Logical rectangles are not adjacent there | Scale, transform, and positions |
| Outputs overlap or move unexpectedly | Invalid explicit position or automatic fallback | Niri warning and logical dimensions |
| External display gets a changing connector name | Dock/port enumeration | EDID identity; repeated output records |
| Two identical monitors receive the wrong rule | Duplicate or unreliable EDID fields | Connector path and private EDID record |
| UI is blurry only in an X11 application | Xwayland/mixed-scale application boundary | Confirm native versus X11; move between scales |
| All applications are too large | Output scale or global toolkit override | Niri scale; environment variables |
| Lid closes and laptop suspends despite external display | logind docked detection or effective policy | Connected outputs; merged logind config; logind journal |
| Lid closes and laptop never suspends when alone | Inhibitor or logind policy | `systemd-inhibit --list`; logind state and journal |
| Laptop continues but internal panel remains logically active with lid shut | Niri/kernel lid detection | Niri output report; input switch state; Niri journal |
| External image works but sound stays internal | WirePlumber route/default sink | `wpctl status` |
| Workspaces seem to disappear after unplug | They migrated to another output | `niri msg workspaces`; overview |
| Screen stays black after an `off` rule | Persistent output rule | TTY3; remove host override; validate |
| Screens do not wake after idle | swayidle action/session IPC | swayidle process; Niri user journal; input activity |
| Layout changes after every hot-plug | Automatic placement or conditional topology needed | Record connection set; consider Kanshi only if repeated |

### Inspect the active Stow selection

Once host packages exist:

```bash
readlink -f ~/.config/niri/config.kdl
readlink -f ~/.config/niri/host.kdl
sed -n '1,200p' ~/.config/niri/host.kdl
niri validate
```

The resolved `host.kdl` path must point to the intended single package. A
regular local file is not automatically wrong, but it is outside the selected
reproducible design and must be reconciled deliberately.

Do not use `stow --adopt` to absorb an unexplained local host file. Review its
content and provenance before deciding whether it belongs in a tracked profile
or a private record.

### Do not use `xrandr` as the Wayland authority

`xrandr` talks to an X server. Under Niri, it can describe the Xwayland
compatibility environment seen by legacy applications, not configure Niri's
physical DRM outputs. A plausible `xrandr` result does not override
`niri msg outputs`.

Use Niri's configuration and IPC for Niri outputs. Use lower-level DRM tools
only when diagnosing why the kernel did not expose expected hardware.

## Recovery

### Remove a bad host override

From the open TTY3 recovery shell, save any remaining work if possible. Then
remove the selected host package link using its exact profile name:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --dir=hosts --delete --verbose --target="$HOME" t14-r7
```

Because the include is optional, the shared configuration will fall back to
automatic output behavior when `host.kdl` disappears. Validate the deployed
base:

```bash
niri validate --config ~/.config/niri/config.kdl
```

If the graphical session remains unusable, disable greetd and test Niri
manually as documented in guide 16:

```bash
sudo systemctl disable --now greetd.service
niri-session -l
```

The `hosts/` packages do not exist yet, so these are future rollback commands,
not actions to perform on the current repository.

### Recover a live session whose only output was turned off

Do not guess `WAYLAND_DISPLAY` or `NIRI_SOCKET` from TTY. Edit or remove the
known host file, run offline configuration validation, and then return to the
session or restart it deliberately. If necessary, stop greetd after saving
work and use the manual session path.

### Hardware fallback

If neither automatic Niri output selection nor the Linux console shows an
image, the problem may be below the Niri configuration:

- cable, adapter, dock, or input selection;
- panel or backlight hardware;
- kernel DRM/amdgpu initialization;
- firmware or GPU state;
- unsupported link bandwidth;
- a failed resume.

Inspect the current boot:

```bash
journalctl -b -k --no-pager | grep -Ei 'drm|amdgpu|edid|display|connector|link'
journalctl --user -b -u niri.service --no-pager
```

Do not add a custom mode, render-device debug option, or static device group
merely because an external cable produced a blank screen. Match the remedy to
the layer that failed.

## Alternatives and trade-offs

| Strategy | Advantage | Cost | Project choice |
| --- | --- | --- | --- |
| No output blocks | Maximum portability and robust first boot | Scale/placement may not match preference | Current baseline |
| One shared output block | Simple | Assumes both panels and preferences are identical | Rejected |
| Local untracked `host.kdl` | Easy and private | Git cannot reconstruct it; relies only on backup | Fallback for sensitive identity, not primary design |
| Explicit per-model Stow host packages | Reproducible, reviewable, selected intentionally | Requires two measured profiles and deployment step | Selected future design |
| Hostname-detecting startup script | Automatic | Hidden branching, hostname coupling, more failure paths | Rejected |
| Kanshi profiles | Good for connection-dependent topologies | Extra daemon and configuration owner | Deferred until repeated topology requires it |
| Another profile daemon | May offer richer matching | More dependencies and another policy surface | No current need |

The selected design is deliberately boring: shared behavior stays shared;
measured hardware differences become tiny explicit packages; dynamic profile
software is added only for a topology that static rules cannot express well.

## Decisions recorded by this guide

- The shared Niri configuration remains free of active output blocks today.
- The two T14 Gen 1 AMD laptops are not assumed to contain identical panels.
- `1920x1080@60.049`, `1920x1080@48.040`, and the scale 1.5 experiment are
  evidence from one machine, not values for both.
- Exact refresh digits come from `niri msg outputs`; custom modes and modelines
  are not used for advertised laptop timings.
- Scale is chosen from ergonomics and application testing after measuring each
  machine, not from PPI alone.
- Output positions use logical dimensions after scale and transform.
- Niri's automatic placement remains the external-display baseline until a
  fixed physical arrangement exists.
- Niri owns internal-panel lid on/off; systemd-logind owns the suspend decision.
- Closing the lid alone suspends; docked or multi-display lid close continues
  on the external display under the existing logind policy.
- Temporary idle monitor power uses Niri actions, not persistent `output off`.
- Niri workspace migration is tested during unplug/reconnect before adding
  automation.
- Future host profiles will be explicit `hosts/t14-r5` and `hosts/t14-r7` Stow
  packages feeding one optional `~/.config/niri/host.kdl` include.
- No hostname-detection script or public EDID serial is part of the design.
- Kanshi remains deferred until a repeated connection-dependent topology
  demonstrates the need for profiles.
- The possible 48 Hz battery mode remains a measured future experiment and is
  not coupled to TLP today.

## Completion checklist

- [ ] Both ThinkPads have separate private output records.
- [ ] Connector, EDID metadata, exact modes, scale, and logical dimensions are recorded.
- [ ] Hardware serials remain outside the public repositories.
- [ ] Scale candidates have been tested with native Wayland and XWayland applications.
- [ ] The chosen refresh mode has passed repeated suspend/resume tests.
- [ ] Any claimed 48 Hz battery benefit is based on controlled measurement.
- [ ] Logical coordinates are calculated after scale and transform.
- [ ] Cursor adjacency matches the physical monitor arrangement.
- [ ] Hot-plug preserves access to all windows and workspaces.
- [ ] Reconnection restores external workspaces where expected.
- [ ] External audio is tested separately through PipeWire/WirePlumber.
- [ ] Lid-close behavior passes alone, on AC, on battery, and with an external display.
- [ ] No duplicate lid or display-power script competes with Niri and logind.
- [ ] The portable base remains valid with no host file.
- [ ] Only one future host Stow package is deployed per machine.
- [ ] Kanshi remains absent unless a documented repeated topology needs it.
- [ ] TTY3 and automatic-output fallback are understood before activating overrides.

## Sources

- [Niri: outputs](https://niri-wm.github.io/niri/Configuration:-Outputs.html)
- [Niri: include](https://niri-wm.github.io/niri/Configuration:-Include.html)
- [Niri: configuration introduction](https://niri-wm.github.io/niri/Configuration:-Introduction.html)
- [Niri: switch events](https://niri-wm.github.io/niri/Configuration:-Switch-Events.html)
- [Niri: workspaces](https://niri-wm.github.io/niri/Workspaces.html)
- [Niri FAQ: connection-dependent output configuration](https://niri-wm.github.io/niri/FAQ.html#how-do-i-change-output-configuration-based-on-connected-monitors)
- [Niri: fractional layout](https://niri-wm.github.io/niri/Development:-Fractional-Layout.html)
- [Niri: layout](https://niri-wm.github.io/niri/Configuration:-Layout.html)
- [Niri IPC](https://niri-wm.github.io/niri/IPC.html)
- [kanshi(5)](https://man.archlinux.org/man/kanshi.5.en)
- [systemd-logind.service(8)](https://man.archlinux.org/man/systemd-logind.service.8.en)
- [logind.conf(5)](https://man.archlinux.org/man/logind.conf.5.en)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)

## Next guide

Continue with XDG base and user directories, desktop entries, MIME types and
associations, locale versus keyboard, fonts and font matching, application
defaults, generated caches, and the boundary between portable configuration
and per-user state.
