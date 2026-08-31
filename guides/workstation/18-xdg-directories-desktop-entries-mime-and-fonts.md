# XDG directories, desktop entries, MIME associations, fonts, and locales

## Purpose and scope

A graphical application does not integrate with the workstation merely because
its executable exists. Several independent conventions answer different
questions:

- where the application stores configuration, data, state, cache, and runtime
  objects;
- where the user's Documents, Downloads, Pictures, and other well-known
  directories are located;
- which language, regional formats, and keyboard mapping the session uses;
- how a launcher discovers and presents the application;
- how the system determines the type of a file;
- which installed application should open that type or a URL scheme;
- which font family supplies each requested glyph.

Most of these conventions come from freedesktop.org and are commonly described
as “XDG”. They share paths and terminology, but they are not one mechanism. An
XDG portal does not choose the default PDF reader, an XDG user directory is not
an XDG base directory, and a locale does not select the physical keyboard.

This guide explains those boundaries and records the decisions already used by
the two ThinkPad installations. It does not add packages or change the running
desktop. The operational steps remain in post-install chapters 08 and 09, while
the reproducible defaults remain in `niri-dotfiles`.

## Current project contract

The published system has this baseline:

| Area | Current decision |
| --- | --- |
| System locale | `LANG=en_US.UTF-8` |
| Console keyboard | `KEYMAP=us` or `KEYMAP=es`, selected per physical keyboard |
| Portable Niri keyboard | `xkb { layout "us" }` until the host-override design is implemented |
| User directories | English names: `Desktop`, `Documents`, `Downloads`, `Music`, `Pictures`, `Public`, `Templates`, and `Videos` |
| User configuration root | Default `$XDG_CONFIG_HOME`, normally `~/.config` |
| Default applications | Tracked in `mimeapps/.config/mimeapps.list` and deployed by GNU Stow |
| General UI font | Noto Sans |
| Terminal font | Noto Sans Mono |
| Emoji font | Noto Color Emoji |
| Document compatibility | Liberation Sans, Serif, and Mono |
| CJK coverage | Optional `noto-fonts-cjk`, not part of the required baseline |
| Nerd Fonts | Deferred until a real icon or theme requirement is accepted |

The separation is deliberate. One ThinkPad can use a Spanish physical
keyboard while retaining English messages and English directory names. A
future host package can change Niri's XKB layout without changing `LANG`, file
associations, font fallback, or the names of existing directories.

## One integration chain, several owners

Opening a PDF from Nautilus illustrates the complete path:

1. the shared MIME database classifies the file as `application/pdf`;
2. `mimeapps.list` selects `org.gnome.Papers.desktop` as the default;
3. the desktop-entry search path resolves that ID to an installed file;
4. the entry describes how Papers is activated and which metadata to show;
5. Papers creates a native graphical client in the Niri session;
6. a file chooser, when needed, is mediated separately through a portal;
7. Fontconfig resolves each requested font and missing glyph through fallback.

These identifiers must not be conflated:

| Identifier | Example | What consumes it |
| --- | --- | --- |
| Executable | `papers` | Shell or an `Exec=` fallback |
| Desktop-file ID | `org.gnome.Papers.desktop` | Launchers and MIME defaults |
| D-Bus application name | Application-specific `org.*` name | D-Bus activation |
| Wayland application ID | Reported by Niri for a window | Niri window rules |
| MIME type | `application/pdf` | Shared MIME and default-application logic |
| Display name | `Papers` | Human-facing menus and launchers |

They can look similar without being interchangeable. A desktop-file ID is not
automatically a Wayland application ID, and neither is necessarily the command
typed in a shell.

## XDG base directories

### Configuration, data, state, cache, and runtime

The XDG Base Directory Specification gives each kind of per-user material a
separate default home:

| Variable | Default when unset or empty | Intended contents | Persistence |
| --- | --- | --- | --- |
| `XDG_CONFIG_HOME` | `~/.config` | User-editable application policy and preferences | Persistent |
| `XDG_DATA_HOME` | `~/.local/share` | User-specific data such as application resources | Persistent |
| `XDG_STATE_HOME` | `~/.local/state` | Histories and restartable state that is not portable data | Persistent |
| `XDG_CACHE_HOME` | `~/.cache` | Regenerable, non-essential acceleration data | Disposable |
| `XDG_RUNTIME_DIR` | No fallback is defined | Sockets, pipes, locks, and other live-session objects | Login-bound and volatile |

The variables need not be explicitly exported for their documented defaults
to apply. Consequently, an empty result from:

```bash
printf '%s\n' "$XDG_CONFIG_HOME"
```

does not mean that applications have no configuration directory. Inspect the
effective defaults without altering the environment:

```bash
printf 'config  %s\n' "${XDG_CONFIG_HOME:-$HOME/.config}"
printf 'data    %s\n' "${XDG_DATA_HOME:-$HOME/.local/share}"
printf 'state   %s\n' "${XDG_STATE_HOME:-$HOME/.local/state}"
printf 'cache   %s\n' "${XDG_CACHE_HOME:-$HOME/.cache}"
printf 'runtime %s\n' "$XDG_RUNTIME_DIR"
```

Any explicitly configured XDG path must be absolute. A relative value is
invalid under the specification and should be ignored by an implementation.
This project does not globally redefine the defaults: doing so can expose
applications with incomplete support and make troubleshooting unnecessarily
different from a normal Arch installation.

### Search directories and precedence

Two colon-separated variables add system search paths:

| Variable | Default | Meaning |
| --- | --- | --- |
| `XDG_CONFIG_DIRS` | `/etc/xdg` | Administrator and vendor configuration searched after user configuration |
| `XDG_DATA_DIRS` | `/usr/local/share:/usr/share` | Local and distribution data searched after user data |

The user location has higher precedence than the system list. Within each
list, an earlier component has higher precedence. This explains why a file
below `~/.local/share/applications` can override an installed desktop entry
with the same desktop-file ID from `/usr/share/applications`.

Search lists are not merge rules by themselves. Each consuming specification
defines whether same-named content is replaced, combined, or ignored. Read the
program-specific rule before assuming that every XDG file behaves like a
systemd drop-in.

### The runtime directory

`XDG_RUNTIME_DIR` is different from every directory above. In this system,
`pam_systemd` creates the login-bound directory, normally `/run/user/$UID`, and
the graphical session inherits it. It must be owned by the user, private, and
on a local filesystem.

Inspect it inside Niri:

```bash
printf '%s\n' "$XDG_RUNTIME_DIR"
stat -c '%U %G %a %n' "$XDG_RUNTIME_DIR"
find "$XDG_RUNTIME_DIR" -maxdepth 1 -type s -printf '%f\n' | sort
```

The exact socket list changes while applications run. PipeWire, Wayland,
D-Bus, keyring, and other session components can place live objects there.
Never version, restore, synchronize, or manually recreate this directory. Its
contents must not survive a reboot or a complete logout.

### `~/.local/bin` has no `XDG_BIN_HOME`

The base-directory specification permits user executables in
`~/.local/bin`. There is no standard `XDG_BIN_HOME` variable. If scripts are
added there later, they must be reviewed source, executable only when needed,
and present in the session `PATH` deliberately. A script is not portable merely
because it lives below `.local`; compiled executables can also be
architecture-specific.

## Base directories are not user directories

The similar names cause a recurring misunderstanding:

| Mechanism | Examples | Purpose |
| --- | --- | --- |
| XDG base directories | `~/.config`, `~/.local/share`, `~/.cache` | Classify application-owned files by semantics |
| XDG user directories | `~/Documents`, `~/Downloads`, `~/Pictures` | Publish user-facing locations for personal content |

`XDG_DOCUMENTS_DIR` does not replace `XDG_DATA_HOME`. Documents belong to the
user; an application's internal database belongs to its data or state
directory. Applications should query the configured user-directory mapping
instead of assuming a literal English or translated path.

## XDG user directories

### Generated mapping

Post-install chapter 08 ran:

```bash
xdg-user-dirs-update
```

That command created the directories and generated two local files:

```text
~/.config/user-dirs.dirs
~/.config/user-dirs.locale
```

The first maps fixed symbolic names to paths. The second records the locale
used when names were generated so migration tools can notice a later locale
change. The expected map is equivalent to:

```bash
XDG_DESKTOP_DIR="$HOME/Desktop"
XDG_DOWNLOAD_DIR="$HOME/Downloads"
XDG_TEMPLATES_DIR="$HOME/Templates"
XDG_PUBLICSHARE_DIR="$HOME/Public"
XDG_DOCUMENTS_DIR="$HOME/Documents"
XDG_MUSIC_DIR="$HOME/Music"
XDG_PICTURES_DIR="$HOME/Pictures"
XDG_VIDEOS_DIR="$HOME/Videos"
```

This generated file is local machine state, not a Stow package in this
project. The reproducible decision is documented by the post-install procedure;
the resulting path map is inspected on each installation.

### Query paths instead of assuming them

Use fixed, reviewed names with `xdg-user-dir`:

```bash
xdg-user-dir DESKTOP
xdg-user-dir DOWNLOAD
xdg-user-dir DOCUMENTS
xdg-user-dir MUSIC
xdg-user-dir PICTURES
xdg-user-dir PUBLICSHARE
xdg-user-dir TEMPLATES
xdg-user-dir VIDEOS
```

The accepted names are not arbitrary. Do not pass unchecked input as the
argument to `xdg-user-dir`; use a hard-coded value selected by the script or
operator.

When a path is intentionally customized, change the mapping as the ordinary
user and create or migrate the target deliberately. For example:

```bash
xdg-user-dirs-update --set DOWNLOAD "$HOME/Downloads"
```

Changing a line does not safely migrate files already stored in the old
directory. Renaming populated user directories requires a separate plan for
file movement, application bookmarks, backup coverage, and rollback.

### Language does not own existing directory names

The first run can localize names according to the current locale, but changing
`LANG` later must not be treated as permission to rename populated folders.
This project intentionally keeps English directory names even if one machine
uses a Spanish keyboard or a user later chooses Spanish messages.

Do not run `xdg-user-dirs-update --force` casually. Its job is to reset the
mapping and recreate directories, not to understand every file, bookmark, or
application reference that might need migration.

`~/Public` is also only a directory. Its presence does not start a web server,
enable file sharing, open firewalld, or publish anything on the network.

## Locale, language, keyboard, and time zone

### Four independent choices

| Choice | Current owner | Example in this project |
| --- | --- | --- |
| Language and regional conventions | Locale variables and `/etc/locale.conf` | `LANG=en_US.UTF-8` |
| Linux virtual-console keyboard | `/etc/vconsole.conf` | `KEYMAP=us` or `KEYMAP=es` |
| Niri graphical keyboard | Niri XKB configuration | Portable baseline `layout "us"` |
| Civil time zone | `/etc/localtime` | Chosen independently during installation |

A locale affects messages, collation, character classes, dates, numbers, and
other regional conventions. It does not describe the labels printed on the
physical keyboard. A keymap or XKB layout maps physical keys; it does not
translate application messages. A time zone determines civil time; it is not
derived from either setting.

This separation is why the second ThinkPad may eventually receive a Spanish
XKB host override without changing the shared Niri layout or the English
application baseline.

### Generated locales and selected locale

`/etc/locale.gen` selects which locale definitions glibc generates.
`locale-gen` builds those definitions. `/etc/locale.conf` selects the
system-wide default inherited by services and users unless a later layer
overrides it.

Inspect rather than change:

```bash
locale
locale -a
localectl status
cat /etc/locale.conf
cat /etc/vconsole.conf
```

The canonical result contains `LANG=en_US.UTF-8`. An optional generated
`es_ES.UTF-8` locale may coexist without becoming the default. Installing the
`hunspell-es_es` dictionary likewise adds Spanish spelling data; it changes
neither `LANG` nor the keyboard.

### Locale-variable precedence

For a locale category such as time or messages, the effective selection is:

1. a non-empty `LC_ALL`, if present;
2. the category-specific variable, such as `LC_TIME` or `LC_MESSAGES`;
3. `LANG` as the fallback.

Useful categories include:

| Variable | Examples of affected behavior |
| --- | --- |
| `LC_CTYPE` | Character classification and case conversion |
| `LC_COLLATE` | Sorting and comparison order |
| `LC_MESSAGES` | Translated messages and desktop-entry localized names |
| `LC_TIME` | Date and time formatting |
| `LC_NUMERIC` | Decimal formatting |
| `LC_MONETARY` | Currency formatting |

Do not set `LC_ALL` globally. `locale.conf(5)` does not permit it in
`/etc/locale.conf`, and a persistent value would suppress all more specific
choices. A deterministic one-command environment is different:

```bash
LC_ALL=C sort input.txt
```

That override ends with the command. It does not mutate the user session.

Locale changes normally apply to newly created login sessions. Editing a file
and then testing in an old Kitty window can therefore produce misleading
results. Verify the environment in the process that actually launches the
application.

## Desktop entries

### Metadata for launchers and integration

A desktop entry is a UTF-8 key file, usually ending in `.desktop`. For an
application, its main group is `[Desktop Entry]` and `Type=Application`.
Launchers such as Fuzzel discover these entries and present their localized
name, icon, keywords, and actions.

Typical locations are:

| Scope | Default location |
| --- | --- |
| Current user | `~/.local/share/applications` |
| Locally installed system software | `/usr/local/share/applications` |
| Distribution packages | `/usr/share/applications` |

Packages own the entries in `/usr/share/applications`. Do not edit them: a
package upgrade may replace the file, and the mutation cannot be reproduced on
the other machine.

### Desktop-file IDs

The ID is derived from the path below the `applications` directory, replacing
subdirectory separators with hyphens. For example:

```text
/usr/share/applications/org.gnome.Papers.desktop
                            └─ org.gnome.Papers.desktop
```

An entry at the user level with the same ID has higher search precedence than
the system entry. This makes per-user customization possible, but also makes a
stale local copy capable of silently shadowing a newer packaged file.

Before blaming the package, locate every matching ID:

```bash
find "${XDG_DATA_HOME:-$HOME/.local/share}/applications" \
    /usr/local/share/applications \
    /usr/share/applications \
    -name 'org.gnome.Papers.desktop' -print 2>/dev/null
```

Do not copy a complete system desktop file into the user directory merely to
change one visual detail. If a local override is genuinely required, keep it
small but complete and valid, review the consequences of shadowing, and record
ownership in `niri-dotfiles`; desktop entries with the same ID do not merge.

### Important keys

| Key | Meaning |
| --- | --- |
| `Type` | `Application`, `Link`, or `Directory`; application launchers use `Application` |
| `Name` | Human-facing name; localized forms such as `Name[es]` may coexist |
| `GenericName` and `Comment` | Additional human-facing description |
| `Icon` | Icon name resolved through the icon theme, or an absolute path |
| `Exec` | Executable and arguments used when normal command activation applies |
| `TryExec` | Optional executable-presence check that can hide an unavailable entry |
| `Terminal` | Whether the command needs a terminal window |
| `MimeType` | Semicolon-separated types the application declares it can handle |
| `Categories` and `Keywords` | Launcher classification and search metadata |
| `NoDisplay` | Keep the entry usable but omit it from ordinary menus |
| `Hidden` | Treat the entry as deleted at that precedence level |
| `OnlyShowIn` and `NotShowIn` | Limit menu visibility by desktop environment |
| `Actions` | Additional named launcher actions |
| `DBusActivatable` | Request D-Bus activation instead of normal `Exec` activation |

`NoDisplay=true` does not disable an application. It can still be a MIME
handler or activation target. `Hidden=true` is stronger: the specification
treats it as deletion at that level. A user entry with the same ID and
`Hidden=true` can mask a system entry, but use that only as a documented,
reversible decision.

### `Exec=` is not a shell script

`Exec=` contains an executable followed by arguments. It does not acquire
shell pipelines, redirections, `$()` substitution, wildcard expansion, or
environment-assignment syntax merely because those constructs work in Bash.
Quoting and escaping follow the Desktop Entry Specification, not a copied shell
command.

The launcher expands defined field codes:

| Code | Expansion |
| --- | --- |
| `%f` | One local file |
| `%F` | Multiple local files, each as a separate argument |
| `%u` | One URL, which may represent a local file |
| `%U` | Multiple URLs, each as a separate argument |
| `%i` | Icon option derived from `Icon=` |
| `%c` | Localized application name |
| `%k` | Location of the desktop file |
| `%%` | Literal percent sign |

Only one of `%f`, `%F`, `%u`, or `%U` may appear. Multi-value codes must stand
as their own argument. Deprecated or invented codes do not make a valid entry.

Avoid `Exec=sh -c ...` unless the shell is itself a reviewed and necessary part
of the design. It creates another quoting and code-execution boundary. For a
complex reusable action, prefer a small reviewed executable script and a
simple desktop entry that calls it.

When `DBusActivatable=true`, compliant launchers should activate the
application over D-Bus and ignore `Exec`; the entry should still retain a
compatible `Exec` fallback. Therefore, editing only `Exec` may appear to have
no effect on a D-Bus-activated application.

### Application entry versus autostart entry

Both mechanisms use the desktop-entry format but search different locations
and answer different questions:

| Location | Meaning |
| --- | --- |
| `~/.local/share/applications/*.desktop` | Discoverable user applications |
| `/usr/share/applications/*.desktop` | Discoverable packaged applications |
| `~/.config/autostart/*.desktop` | User-session autostart |
| `/etc/xdg/autostart/*.desktop` | System-provided user-session autostart |

Installing an application entry does not automatically start its process at
login. Conversely, the `udiskie.desktop` file tracked in the project's
`autostart` Stow package is a session-start instruction, not the MIME identity
used to open directories.

Systemd user activation, D-Bus activation, XDG autostart, and a Niri
`spawn-at-startup` line are also separate owners. Guide 14 explains the session
architecture; use one owner for each long-lived component.

### Inspect and validate

Inspect the packaged file without editing it:

```bash
desktop-file-validate /usr/share/applications/org.gnome.Papers.desktop
grep -E '^(Type|Name|Exec|TryExec|DBusActivatable|Terminal|MimeType|NoDisplay|Hidden)=' \
    /usr/share/applications/org.gnome.Papers.desktop
```

Validate any deliberate user entry before deploying it:

```bash
desktop-file-validate ~/.local/share/applications/example.desktop
```

After adding or removing user application entries that advertise MIME types,
rebuild that user's application MIME cache:

```bash
update-desktop-database ~/.local/share/applications
```

The resulting `mimeinfo.cache` is generated state. Do not version it. Pacman
hooks maintain the corresponding system cache for packaged entries.

## MIME types and the shared database

### Classification is not selection

MIME integration has three distinct questions:

| Question | Owner |
| --- | --- |
| What type is this file? | Shared MIME-info database |
| Which applications claim to support the type? | Desktop entries plus added/removed associations |
| Which one should open it by default? | `[Default Applications]` in `mimeapps.list` |

Changing the default application does not redefine the file type. Adding a new
type does not choose a handler. Installing a handler does not necessarily make
it the default.

### Type detection uses names and contents

A MIME type has a media and subtype, for example:

```text
text/plain
image/png
application/pdf
x-scheme-handler/https
```

The shared database can use filename glob patterns, literal names, content
“magic”, inheritance, aliases, and XML namespace rules. An extension is useful
evidence, not the complete model. Conflicting filename patterns can require
content inspection.

Inspect a real file with more than one interface:

```bash
xdg-mime query filetype "$HOME/Documents/example.pdf"
file --brief --mime-type "$HOME/Documents/example.pdf"
gio info -a standard::content-type "$HOME/Documents/example.pdf"
```

Replace the path with an existing non-sensitive test file. Run `xdg-mime`
inside the desktop session and never through `sudo`.

Packages install shared MIME source XML below `/usr/share/mime/packages` and
hooks generate the optimized database below `/usr/share/mime`. User-defined
types, if ever required, belong under
`~/.local/share/mime/packages` and require:

```bash
update-mime-database ~/.local/share/mime
```

Do not edit generated files such as `globs2`, `magic`, or `mime.cache`
directly. The current project defines no custom MIME type, so it needs no user
database source or generated cache in Git.

### URL schemes use handler pseudo-types

HTTP and HTTPS defaults are expressed as:

```text
x-scheme-handler/http
x-scheme-handler/https
```

They are not ordinary disk-file formats, but the same association machinery
selects a desktop-file ID. Other schemes such as `mailto` should remain
unassigned until a corresponding application and account policy are selected.
The project currently has no canonical email client.

## `mimeapps.list`

### Lookup order

The complete standard search order begins with the most specific user
override:

1. `$XDG_CONFIG_HOME/$desktop-mimeapps.list`;
2. `$XDG_CONFIG_HOME/mimeapps.list`;
3. each `$XDG_CONFIG_DIRS/$desktop-mimeapps.list`;
4. each `$XDG_CONFIG_DIRS/mimeapps.list`;
5. `$XDG_DATA_HOME/applications/$desktop-mimeapps.list` (deprecated);
6. `$XDG_DATA_HOME/applications/mimeapps.list` (deprecated);
7. each `$XDG_DATA_DIRS/applications/$desktop-mimeapps.list`;
8. each `$XDG_DATA_DIRS/applications/mimeapps.list`.

`$desktop` is derived from lower-cased components of
`XDG_CURRENT_DESKTOP`. Desktop-specific files are advanced overrides and may
specify defaults, but not added or removed associations.

The project's tracked path is the recommended generic user location:

```text
~/.config/mimeapps.list
```

It is a symlink created by Stow to:

```text
mimeapps/.config/mimeapps.list
```

No deprecated compatibility copy or second symlink is added. If a particular
application is later proven to write only a deprecated location, document the
observed behavior before introducing another path.

### Sections and separators

The format supports:

```ini
[Default Applications]
application/pdf=org.gnome.Papers.desktop;

[Added Associations]
application/pdf=org.gnome.Papers.desktop;

[Removed Associations]
application/pdf=some.unwanted.Reader.desktop;
```

Values are semicolon-separated desktop-file IDs. Keep the trailing semicolon.

- `Default Applications` gives the preferred handlers, in order;
- `Added Associations` says an application supports a type in addition to its
  desktop entry;
- `Removed Associations` suppresses a declared association.

Default selection and the complete “Open With” list are related but not the
same algorithm. A default must identify an installed, valid desktop entry that
is associated with the requested type. The current project needs only
`[Default Applications]`; it does not invent support claims already provided
by the installed application entries.

### Current reproducible defaults

The tracked map groups multiple MIME types under a small application set:

| Role | Desktop-file ID | Types covered |
| --- | --- | --- |
| Browser | `firefox.desktop` | HTTP, HTTPS, and HTML |
| File manager | `org.gnome.Nautilus.desktop` | Directories |
| Document viewer | `org.gnome.Papers.desktop` | PDF |
| Image viewer | `org.gnome.Loupe.desktop` | JPEG, PNG, WebP, GIF, and TIFF |
| Text editor | `org.gnome.TextEditor.desktop` | Plain text and Markdown |
| Media player | `io.github.celluloid_player.Celluloid.desktop` | MP4, Matroska, WebM, MP3, FLAC, and Ogg audio |
| Archive manager | `org.gnome.FileRoller.desktop` | ZIP, 7-Zip, and tar archives |
| Calendar | `org.gnome.Calendar.desktop` | `text/calendar` files such as `.ics` |
| Writer | `libreoffice-writer.desktop` | ODT, DOC, and DOCX |
| Calc | `libreoffice-calc.desktop` | ODS, XLS, and XLSX |
| Impress | `libreoffice-impress.desktop` | ODP, PPT, and PPTX |

The canonical file itself remains the precise source of truth. Inspect it and
its deployment:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
cat mimeapps/.config/mimeapps.list
ls -l ~/.config/mimeapps.list
readlink -f ~/.config/mimeapps.list
```

The resolved path must be inside the local `niri-dotfiles/mimeapps` package and
the Git tree should be clean.

### Query and open

Check representative defaults:

```bash
xdg-mime query default x-scheme-handler/https
xdg-mime query default inode/directory
xdg-mime query default application/pdf
xdg-mime query default image/png
xdg-mime query default text/plain
xdg-mime query default video/mp4
xdg-mime query default application/zip
xdg-mime query default text/calendar
xdg-mime query default application/vnd.oasis.opendocument.text
```

Expected output, in the same order:

```text
firefox.desktop
org.gnome.Nautilus.desktop
org.gnome.Papers.desktop
org.gnome.Loupe.desktop
org.gnome.TextEditor.desktop
io.github.celluloid_player.Celluloid.desktop
org.gnome.FileRoller.desktop
org.gnome.Calendar.desktop
libreoffice-writer.desktop
```

`gio mime TYPE` shows the registered default and other handlers known through
GLib:

```bash
gio mime application/pdf
gio mime inode/directory
gio mime text/calendar
```

Functional dispatch uses the preferred opener:

```bash
xdg-open https://archlinux.org/
xdg-open "$(xdg-user-dir DOWNLOAD)"
```

Run these inside Niri as the ordinary user. `xdg-open` is a desktop-session
tool and must not be used as root. Also note a subtle diagnostic boundary:
`xdg-mime query default` and the application actually selected by `xdg-open`
can differ when `xdg-open` delegates to a desktop-specific opener. Confirm both
the query and a functional open before declaring the path healthy.

### The Stow and Git consequence

Because `~/.config/mimeapps.list` is a symlink into a Git repository, a
graphical “Make default” button or this command:

```bash
xdg-mime default org.gnome.Loupe.desktop image/png
```

may edit the tracked source directly. That is useful for reproducibility, but
it also means an exploratory click can dirty the repository.

After changing a default:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short
git diff -- mimeapps/.config/mimeapps.list
```

Keep and commit only an intentional, portable policy. Restore an accidental
change through Git before continuing. Do not run `sudo xdg-mime`; system-wide
ownership is not the project's design.

### Portals do not own default applications

An XDG Desktop Portal can mediate a file chooser, screenshots, screencasts,
URI opening, or sandbox-visible document grants. That is a security and desktop
integration boundary. It does not replace the shared MIME database or make the
portal backend the default PDF reader.

The simplified split is:

| Task | Owner |
| --- | --- |
| Select or grant access to a file | Portal interface and backend when used |
| Determine a local file's type | Shared MIME database |
| Choose its normal handler | `mimeapps.list` and desktop entries |
| Authorize a privileged action | polkit, not the portal or MIME layer |

Guide 10 explains portals and polkit in detail.

## Fonts and Fontconfig

### Files, families, faces, and fallback

A font file can contain one or more faces. Applications normally request a
family plus style and properties, not a filename. Fontconfig searches known
fonts, applies configuration and aliases, and returns the best match followed
by fallback candidates for glyphs the first face lacks.

Useful terms are:

| Term | Meaning |
| --- | --- |
| Font file | Installed `.ttf`, `.otf`, or other supported resource |
| Family | Human-facing group such as `Noto Sans` |
| Face | A family member such as Regular, Bold, or Italic |
| Generic family | Alias such as `sans-serif`, `serif`, or `monospace` |
| Glyph | The visual shape used for a character |
| Fallback | A later font selected when the preferred font lacks a glyph |

The filename is therefore not a reliable string to paste into Kitty, Waybar,
or Fuzzel. Query the family reported by Fontconfig.

### Installation paths

The normal paths are:

| Scope | Location |
| --- | --- |
| Packaged/system fonts | `/usr/share/fonts` |
| Per-user fonts | `~/.local/share/fonts` |
| Deprecated per-user path | `~/.fonts` |
| Per-user Fontconfig policy | `~/.config/fontconfig/fonts.conf` or `conf.d/*.conf` |
| Generated per-user cache | `~/.cache/fontconfig` |

Prefer official packages for the shared baseline. They provide ownership,
updates, removal, and pacman hooks. Before adding a manually downloaded font,
verify its license, provenance, exact family names, and whether both machines
may legally receive it. Do not commit proprietary font binaries casually.

### Current project families

Post-install chapter 08 installs:

```text
noto-fonts
noto-fonts-emoji
ttf-liberation
```

Chapter 05 already supplies DejaVu through the graphical stack. The current
dotfiles request:

| Component | Request |
| --- | --- |
| Kitty | `Noto Sans Mono` |
| Waybar | `Noto Sans`, then generic `sans-serif` |
| Fuzzel | `Noto Sans` |
| Mako | `Noto Sans` |
| swaylock | `Noto Sans` |

Noto supplies broad general text coverage; Noto Color Emoji supplies color
emoji; Liberation supplies metric-compatible document families. CJK coverage
is optional because `noto-fonts-cjk` is substantially larger and is not needed
for the canonical English and Spanish use case.

The baseline deliberately does not require a Nerd Font. Current Waybar labels
work without making private-use icon glyphs a dependency. If the accepted
future theme or modular shell needs icon glyphs, select the minimum maintained
package, record the exact family in dotfiles, and verify fallback then. Do not
install every Nerd Font preemptively.

### Inspect selection and coverage

List known families compactly:

```bash
fc-list : family | sort -fu | less
```

Ask Fontconfig for the best match:

```bash
fc-match sans-serif
fc-match serif
fc-match monospace
fc-match 'Noto Sans'
fc-match 'Noto Sans Mono'
fc-match 'Noto Color Emoji'
fc-match 'Liberation Sans'
```

Inspect the ordered fallback list when a glyph problem is suspected:

```bash
fc-match --sort 'Noto Sans' | head -n 20
```

Test representative text in Kitty and graphical applications:

```bash
printf '%s\n' 'English — Español: áéíóú ñ ¿¡ — € → ✓ 😀'
```

A missing square can mean absent glyph coverage, an application-specific
rendering problem, an incorrectly spelled family, or a stale application
process. It does not automatically justify a global `fonts.conf` override.

### Caches are generated

Pacman font hooks normally update system caches when packages change.
Fontconfig-aware applications may also maintain per-user cache under
`~/.cache/fontconfig`. If a manually installed user font is not discovered,
rebuild the relevant cache:

```bash
fc-cache -f ~/.local/share/fonts
```

Then restart the affected application and confirm with `fc-match` or
`fc-list`. Do not track cache files in Git or copy them between the ThinkPads;
they describe local files and are recreated by Fontconfig.

The same principle applies to thumbnails below `~/.cache/thumbnails` and
application MIME caches such as `mimeinfo.cache`: version the source policy,
not its acceleration artifacts.

## Portable policy versus generated and private state

An XDG path does not decide whether its contents belong in Git. Semantics,
sensitivity, portability, and ownership decide.

| Example | Project treatment | Reason |
| --- | --- | --- |
| `~/.config/niri/config.kdl` | Track through Stow | Deliberate portable policy |
| `~/.config/mimeapps.list` | Track through Stow | Deliberate default-application map |
| `~/.config/autostart/udiskie.desktop` | Track through Stow | Deliberate session behavior |
| `~/.config/user-dirs.dirs` | Generate and inspect locally | Locale/path result, not a current Stow package |
| `~/.config/user-dirs.locale` | Do not track | Generated migration marker |
| `~/.local/share/wallpapers` | Track only deliberately selected public assets | Portable user data can be configuration input |
| `~/.local/share/applications/*.desktop` | Track only a reviewed custom entry | User override can shadow package metadata |
| `~/.local/share/fonts` | Do not copy blindly | Licensing, size, provenance, and host needs vary |
| `~/.local/share/recently-used.xbel` | Do not track | Personal activity history |
| `~/.local/share/Trash` | Do not track | Deleted user content and metadata |
| `~/.cache/fontconfig` | Do not track | Regenerable cache |
| `~/.cache/thumbnails` | Do not track | Regenerable previews that can reveal file history |
| `~/.local/share/applications/mimeinfo.cache` | Do not track | Generated application MIME cache |
| `$XDG_RUNTIME_DIR` | Never track or back up | Login-bound sockets and live runtime objects |
| Browser profiles, keyrings, tokens, and account data | Keep out of public repositories | Private state and credentials |

`XDG_STATE_HOME` is designed for persistent state, but persistence does not
make state portable. Histories, window state, recent files, undo data, and logs
often belong in backups only if the backup policy deliberately includes them;
they do not belong in public dotfiles by default.

Guide 19 will define the backup boundary separately from the Git boundary.

## Layered verification

Run the following audit inside the active Niri session as the ordinary user.

### 1. Environment and directories

```bash
locale
localectl status
printf 'XDG_CONFIG_HOME=%s\n' "${XDG_CONFIG_HOME:-$HOME/.config}"
printf 'XDG_DATA_HOME=%s\n' "${XDG_DATA_HOME:-$HOME/.local/share}"
printf 'XDG_STATE_HOME=%s\n' "${XDG_STATE_HOME:-$HOME/.local/state}"
printf 'XDG_CACHE_HOME=%s\n' "${XDG_CACHE_HOME:-$HOME/.cache}"
printf 'XDG_RUNTIME_DIR=%s\n' "$XDG_RUNTIME_DIR"
cat ~/.config/user-dirs.dirs
cat ~/.config/user-dirs.locale
```

Confirm `LANG=en_US.UTF-8`, the deliberate console keymap, absolute effective
base paths, a private runtime directory, and the English user-directory map.

### 2. Desktop entries

```bash
for entry in \
    firefox.desktop \
    org.gnome.Nautilus.desktop \
    org.gnome.Papers.desktop \
    org.gnome.Loupe.desktop \
    org.gnome.TextEditor.desktop \
    io.github.celluloid_player.Celluloid.desktop \
    org.gnome.FileRoller.desktop \
    org.gnome.Calendar.desktop \
    libreoffice-writer.desktop; do
    test -r "/usr/share/applications/$entry" || printf 'missing: %s\n' "$entry"
done
```

No `missing:` line should appear. Validate a suspicious entry individually
rather than changing all caches first.

### 3. MIME defaults

```bash
ls -l ~/.config/mimeapps.list
readlink -f ~/.config/mimeapps.list
xdg-mime query default x-scheme-handler/https
xdg-mime query default inode/directory
xdg-mime query default application/pdf
xdg-mime query default image/png
xdg-mime query default text/calendar
```

Compare every result to the tracked map. If a query is wrong, inspect
precedence and desktop-entry validity before rewriting the file.

### 4. Font selection

```bash
fc-match 'Noto Sans'
fc-match 'Noto Sans Mono'
fc-match 'Noto Color Emoji'
fc-match 'Liberation Sans'
```

Each result must resolve to an installed family appropriate to the request.

### 5. Repository cleanliness

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
```

The tree should be clean. A change to `mimeapps.list` after application tests
must be reviewed as a configuration change, not dismissed as harmless cache.

## Troubleshooting by symptom

| Symptom | Inspect first | Likely boundary |
| --- | --- | --- |
| Application missing from Fuzzel | Desktop entry path, `NoDisplay`, `Hidden`, `TryExec`, `OnlyShowIn`, validation | Desktop-entry discovery |
| Correct app exists but wrong app opens a PDF | File MIME type, `mimeapps.list` precedence, desktop ID validity | MIME default |
| `xdg-mime` reports the expected app but `xdg-open` differs | Functional opener, desktop-specific delegation, session environment | Opener implementation |
| A “Make default” click dirties Git | Diff of tracked Stow source | Intended symlink behavior |
| Desktop-entry `Exec` change appears ignored | `DBusActivatable`, shadowing user entry, running process | Activation mode or precedence |
| Spanish keyboard produces US symbols | Niri XKB or console keymap, depending on where it happens | Input mapping, not locale |
| Menus remain English after a locale change | New login environment, `LC_MESSAGES`, generated locale | Locale inheritance |
| Downloads opens an unexpected path | `user-dirs.dirs` and `xdg-user-dir DOWNLOAD` | XDG user directories |
| Emoji or symbols appear as boxes | `fc-match`, fallback order, package coverage | Font selection |
| Newly installed font is absent | Font path, license/source, `fc-list`, cache, application restart | Font discovery |
| Repeated stale app in menus | User shadow entry and generated `mimeinfo.cache` | Desktop-entry precedence/cache |
| Portal chooser works but default app is wrong | Portal logs separately from MIME query | Two independent layers |

### Debug MIME lookup without editing it

The installed `xdg-mime` implementation can show which files it searches:

```bash
XDG_UTILS_DEBUG_LEVEL=2 xdg-mime query default application/pdf
```

Use the output to find a higher-precedence `mimeapps.list`. Do not delete every
candidate or rebuild unrelated caches blindly.

### Detect a user desktop-entry shadow

If a packaged application behaves differently from its current desktop file:

```bash
find "${XDG_DATA_HOME:-$HOME/.local/share}/applications" \
    /usr/local/share/applications \
    /usr/share/applications \
    -name 'DESKTOP-ID.desktop' -print 2>/dev/null
```

Replace the placeholder with the exact ID. Compare every result. A user copy
may be older than the installed package even though its timestamp looks newer.

### Diagnose a file before changing its handler

```bash
test_file="$HOME/Documents/example-file"
xdg-mime query filetype "$test_file"
file --brief --mime-type "$test_file"
```

Replace the example with an existing safe path. If the classification is
wrong, changing `mimeapps.list` only assigns an application to the wrong type;
it does not repair detection.

## Safe changes and recovery

### Restore the tracked default map

Inspect an accidental change from the repository root:

```bash
git status --short
git diff -- mimeapps/.config/mimeapps.list
```

If the modification is unwanted and uncommitted:

```bash
git restore -- mimeapps/.config/mimeapps.list
```

Because the live file is a symlink, restoring the tracked source restores the
deployed content immediately. Query the default again. Do not remove the
symlink and leave an unreviewed replacement file behind.

To remove the entire deployed map while preserving the repository source:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --delete --verbose --target="$HOME" mimeapps
```

Re-deploy only after checking for a conflicting target:

```bash
stow --simulate --verbose --no-folding --target="$HOME" mimeapps
stow --verbose --no-folding --target="$HOME" mimeapps
```

### Recover from a bad user desktop entry

Do not edit the packaged file. Identify the higher-precedence user entry,
preserve a copy outside the active `applications` search directory, and then
remove or correct only that reviewed override. Revalidate it and rebuild only
the user application cache:

```bash
desktop-file-validate ~/.local/share/applications/example.desktop
update-desktop-database ~/.local/share/applications
```

If the override is owned by a future Stow package, change or unstow the package
rather than deleting its deployed symlink without updating the repository.

### Recover from a bad font override

First confirm whether a user Fontconfig file exists:

```bash
find ~/.config/fontconfig -maxdepth 2 -type f -print 2>/dev/null
fc-match sans-serif
fc-match monospace
```

The current baseline creates no user `fonts.conf`. If an experimental file
caused the regression, move only that known file out of the active directory,
refresh the user cache, and restart the affected application. Do not erase all
of `~/.config` or `/etc/fonts`, and never edit `/etc/fonts/fonts.conf`, which
belongs to the package.

### Recover from a locale mistake

Keep a root TTY available. Confirm that the desired locale is generated, read
`/etc/locale.conf`, and inspect the live environment separately:

```bash
locale -a
cat /etc/locale.conf
locale
```

Remove an unintended per-user or shell override at its real source, then start
a new login session. Do not work around one program by exporting `LC_ALL`
globally.

### Recover from an incorrect user-directory mapping

Read the current map and locate the actual files before editing anything:

```bash
cat ~/.config/user-dirs.dirs
xdg-user-dir DOWNLOAD
find "$HOME" -maxdepth 2 -type d \( -name Downloads -o -name Descargas \)
```

Change the mapping and move content as two deliberate operations. A forced
reset is not a substitute for a migration or a backup.

## Decisions recorded for this project

- Keep the normal XDG base-directory defaults instead of globally relocating
  application trees.
- Treat `XDG_RUNTIME_DIR` as volatile login state and never back it up.
- Keep English XDG user-directory names on both ThinkPads.
- Keep `LANG=en_US.UTF-8` as the canonical interface locale.
- Select console and graphical keyboard layouts independently per physical
  machine.
- Do not set `LC_ALL` globally.
- Use packaged desktop entries in `/usr/share/applications`; do not edit them.
- Keep user default applications in one reviewed
  `~/.config/mimeapps.list` Stow package.
- Treat application “Make default” changes as Git-visible policy changes.
- Keep MIME classification, MIME default selection, portals, polkit, and Niri
  window identity as separate layers.
- Use Noto Sans for the visible desktop, Noto Sans Mono for Kitty, Noto Color
  Emoji for emoji, and Liberation for document compatibility.
- Keep CJK coverage optional and Nerd Fonts deferred until an accepted design
  demonstrates the need.
- Version source configuration, not Fontconfig, thumbnail, MIME, launcher, or
  runtime caches.
- Keep credentials, histories, recent-file records, browser profiles, keyrings,
  and other private state out of public repositories.

## Completion checklist

- [ ] XDG base directories and XDG user directories can be explained as
      different mechanisms.
- [ ] Effective base paths are absolute and the project keeps standard
      defaults.
- [ ] `XDG_RUNTIME_DIR` belongs to the live login, has private permissions, and
      is neither versioned nor restored.
- [ ] `LANG=en_US.UTF-8` remains independent of the console and Niri keyboard
      layouts.
- [ ] English user-directory paths resolve through `xdg-user-dir`.
- [ ] Packaged desktop entries exist and validate without local edits.
- [ ] Desktop-file ID, executable, D-Bus name, and Wayland application ID are
      not treated as synonyms.
- [ ] The `Exec` field and its `%f`, `%F`, `%u`, and `%U` codes are understood
      without assuming Bash syntax.
- [ ] The tracked `mimeapps.list` resolves through the Stow symlink.
- [ ] Representative URL, directory, PDF, image, text, media, archive,
      calendar, and office defaults match the recorded desktop IDs.
- [ ] MIME classification is tested before a handler is changed.
- [ ] Noto Sans, Noto Sans Mono, Noto Color Emoji, and Liberation resolve
      through Fontconfig.
- [ ] Font, MIME, thumbnail, and runtime caches remain untracked.
- [ ] `niri-dotfiles` is clean after functional application tests.
- [ ] Every local override has an identifiable owner and a recovery path.

## Sources

- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir/latest/)
- [xdg-user-dirs upstream overview](https://www.freedesktop.org/wiki/Software/xdg-user-dirs/)
- [xdg-user-dirs-update(1)](https://man.archlinux.org/man/xdg-user-dirs-update.1)
- [xdg-user-dir(1)](https://man.archlinux.org/man/xdg-user-dir.1)
- [Desktop Entry Specification](https://specifications.freedesktop.org/desktop-entry/latest-single/)
- [Desktop Application Autostart Specification](https://specifications.freedesktop.org/autostart/latest/)
- [MIME Applications Associations Specification](https://specifications.freedesktop.org/mime-apps/latest-single/)
- [Shared MIME-info Database Specification](https://specifications.freedesktop.org/shared-mime-info/latest-single/)
- [Thumbnail Managing Standard](https://specifications.freedesktop.org/thumbnail/latest-single/)
- [xdg-mime(1)](https://man.archlinux.org/man/xdg-mime.1)
- [xdg-open(1)](https://man.archlinux.org/man/xdg-open.1)
- [locale.conf(5)](https://man.archlinux.org/man/locale.conf.5)
- [localectl(1)](https://man.archlinux.org/man/localectl.1)
- [vconsole.conf(5)](https://man.archlinux.org/man/vconsole.conf.5)
- [Fontconfig user documentation](https://fontconfig.pages.freedesktop.org/fontconfig/fontconfig-user.html)
- [fc-list(1)](https://man.archlinux.org/man/fc-list.1)
- [fc-match(1)](https://man.archlinux.org/man/fc-match.1)
- [fc-cache(1)](https://man.archlinux.org/man/fc-cache.1)
- [ArchWiki: XDG Base Directory](https://wiki.archlinux.org/title/XDG_Base_Directory)
- [ArchWiki: XDG user directories](https://wiki.archlinux.org/title/XDG_user_directories)
- [ArchWiki: Desktop entries](https://wiki.archlinux.org/title/Desktop_entries)
- [ArchWiki: XDG MIME Applications](https://wiki.archlinux.org/title/XDG_MIME_Applications)
- [ArchWiki: Locale](https://wiki.archlinux.org/title/Locale)
- [ArchWiki: Font configuration](https://wiki.archlinux.org/title/Font_configuration)

## Next step

Continue with guide 19 to explain Restic repositories, what belongs in backup
versus Git, encrypted repository credentials, retention, integrity checks,
restore drills, and recovery media. The first essential edition then closes
with guide 20's journal-led maintenance and incident-recovery workflow.
