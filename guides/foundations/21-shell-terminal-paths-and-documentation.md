# Shell, terminal, paths, redirection, editors, and documentation

## Purpose and scope

The command line is not a collection of incantations. It is an interface made
of several distinct layers: a terminal presents characters, a shell parses a
language, programs receive arguments and byte streams, the kernel manages
processes and file descriptors, and documentation defines the contract of each
command.

This guide builds the model needed to read, understand, and safely adapt the
commands in the runbook, post-install guide, and handbook. It explains:

- the difference between a TTY, terminal emulator, shell, prompt, command,
  process, and session;
- how Bash divides a command line into words and resolves a command name;
- absolute and relative paths, quoting, expansion, globbing, variables, and
  environments;
- standard input, standard output, standard error, pipes, and redirection;
- exit statuses, conditional execution, signals, and interactive job control;
- safe inspection, copying, moving, deletion, searching, and text processing;
- Micro, `sudoedit`, Vim recovery basics, and editor-selection variables;
- Bash startup files and the boundary between shell configuration and the
  graphical session;
- how to read `--help`, manual pages, Info manuals, ArchWiki, package metadata,
  and command examples;
- why commands written for Bash cannot be pasted unchanged into Windows
  PowerShell.

It does not replace a Bash programming book, design a custom prompt, add shell
aliases or plugins, switch the login shell, install another terminal, create a
Bash Stow package, or teach Git. Guide 22 covers Git and GitHub. Guides 01, 02,
03, and 04 remain authoritative for configuration precedence, permissions,
systemd, and package management.

## Current project contract

The installed workstation and the Windows repository workflow intentionally
use different command interpreters:

| Context | Current choice | Responsibility |
| --- | --- | --- |
| Linux virtual console | TTY plus Bash | Login, recovery, and text commands without Niri |
| Niri graphical session | Kitty plus Bash | Terminal window and interactive Linux shell |
| Linux login shell | `/bin/bash` | Default command interpreter recorded for the user |
| Terminal editor | Micro | Canonical approachable editor and recovery editor |
| Manual-page pager | less | Read and search long terminal documentation |
| Windows repository handoff | PowerShell in VS Code | Copy, review, commit, and push project files |
| Completion | `bash-completion` through Arch's Bash setup | Context-aware completion for supported commands |
| Optional search tools | `ripgrep`, `fd`, `fzf`, `jq`, `tree`, and `bat` | Explicit helpers; not aliases replacing standard commands |

Kitty is not Bash. Kitty creates a graphical terminal, displays the character
grid, passes keyboard input to a child program, and renders that program's
output. Bash is normally that child program and interprets the command
language. Opening a different terminal emulator would not turn Bash syntax
into another language; changing the shell would not replace Kitty's window.

The project currently has a portable Kitty package, but no Bash, Micro, or Vim
dotfile package. Bash remains deliberately close to the Arch default:

- no `chsh` operation;
- no aliases that replace `cat`, `grep`, `find`, `ls`, or other standard tools;
- no prompt framework;
- no Fish, Zsh, or Nushell migration;
- no automatic `fzf` keybindings or completion;
- no unreviewed shell code sourced from the network.

This guide explains the stable foundation before those optional choices are
evaluated.

## The layers behind one prompt

### Physical input and virtual consoles

The keyboard produces input events. The kernel and the active graphical or
console stack turn those events into keys and characters according to the
relevant layout. A TTY is a kernel-backed text terminal. On this workstation,
another TTY remains available with a key combination such as `Ctrl+Alt+F2`
when Niri or greetd fails.

The TTY is not a shell. After authentication, the login process starts the
user's configured shell and attaches its standard streams to that terminal.

### Terminal emulator

Kitty is a terminal emulator inside the Wayland session. It implements the
terminal protocol expected by text programs, provides scrollback, clipboard
integration, tabs and windows, and launches a child command. The current
dotfiles define, among other behavior:

- `Ctrl+Shift+C` and `Ctrl+Shift+V` for the regular clipboard;
- `Ctrl+Shift+Enter` for another Kitty window in the current directory;
- `Ctrl+Shift+T` for another tab in the current directory;
- 10,000 lines of scrollback;
- no copy-on-select and no audible bell.

These are terminal mappings. `Ctrl+C` sent without Kitty's `Shift` modifier is
normally input for the foreground terminal program and is commonly interpreted
as an interrupt signal. The two shortcuts are not interchangeable.

### Shell

Bash is a command-language interpreter. It reads text, recognizes shell
syntax, performs expansions and redirections, finds commands, starts programs,
waits for them, and reports an exit status. It also provides builtins such as
`cd`, `type`, `export`, `history`, `jobs`, and `read` that operate on the shell
itself.

### Prompt

The prompt is only the shell's invitation for another command. Documentation
often uses:

```text
$ command run as an ordinary user
# command requiring a root shell
PS C:\> command entered in PowerShell
```

Do not type the leading `$`, `#`, or `PS C:\>` unless the text explicitly says
that it is part of an argument. This handbook normally omits prompts and uses
the code-fence language plus `sudo` to make the intended interpreter and
privilege boundary explicit.

### Command, program, process, and session

These terms describe different things:

| Term | Meaning |
| --- | --- |
| Command line | Text submitted to a shell for parsing |
| Command | A shell construct, function, builtin, or executable invocation |
| Program | Executable code stored on disk or implemented inside the shell |
| Process | A running instance with an ID, environment, working directory, open files, and credentials |
| Job | Bash's interactive representation of one pipeline, possibly containing several processes |
| Session | A broader login or graphical lifetime containing related processes and resources |

The same program may have several processes. A pipeline is one Bash job but
can contain multiple processes. Closing one terminal window is not the same as
logging out of the graphical session.

## Bash and PowerShell are different languages

Both can launch `git`, but they do not parse the surrounding text identically.
The earlier failed multiline `git add` demonstrated this boundary: Bash uses a
backslash at the end of a physical line, whereas PowerShell uses a backtick and
does not recognize backslash as its escape character.

| Intent | Bash on Arch | PowerShell on Windows |
| --- | --- | --- |
| Typical prompt | `$` | `PS D:\path>` |
| Path separator | `/` | Usually `\`, with `/` accepted by many commands |
| Home variable | `$HOME` | `$HOME` or `$Env:USERPROFILE`, with different variable semantics |
| Continue a line explicitly | `\` as the final unquoted character | Backtick as the final character; official guidance prefers natural breaks or one line |
| Discard Unix output | Redirect to `/dev/null` | Redirect the required stream to `$null`, or pipe objects to `Out-Null` |
| Pipeline payload | Primarily byte or text streams | PowerShell objects between PowerShell commands |
| Environment variable | `$PATH` | `$Env:Path` |
| Current directory | `pwd` or `$PWD` | `Get-Location` or `$PWD` |

The safest project handoff rule is simple:

- fenced `bash` commands run on the installed Arch system in Bash;
- fenced `powershell` commands run in the VS Code PowerShell terminal on
  Windows;
- fenced `text` blocks are expected output, messages, or literal content, not
  commands;
- Git commands in downloadable `README-FIRST.md` files stay on one PowerShell
  line when practical.

Do not translate only the path and assume the language around it remains
valid. Redirection, quoting, variables, command substitution, wildcard
expansion, arrays, and pipelines all need interpreter-specific review.

## Anatomy of a simple Bash command

Consider:

```bash
rg --line-number --fixed-strings 'HandleLidSwitch=' /etc/systemd/logind.conf.d
```

After Bash parsing and quote removal, the invoked program receives separate
arguments conceptually equivalent to:

```text
argument 0: rg
argument 1: --line-number
argument 2: --fixed-strings
argument 3: HandleLidSwitch=
argument 4: /etc/systemd/logind.conf.d
```

The parts normally have these roles:

| Part | Role |
| --- | --- |
| `rg` | Command name |
| `--line-number` | Long option |
| `--fixed-strings` | Long option changing pattern interpretation |
| `'HandleLidSwitch='` | Quoted pattern argument |
| `/etc/systemd/logind.conf.d` | Path operand |

Whitespace separates shell words only when it is not quoted or escaped. The
quotes protect syntax during parsing; they are normally removed before `rg`
receives the argument.

Short options may sometimes be combined, but only when that program's manual
permits it. `ls -la` is commonly equivalent to `ls -l -a`; that does not make
every `-abc` valid for every program.

### Placeholders are not literal arguments

Documentation uses placeholders such as `PACKAGE`, `NAME.service`, `BOOT_ID`,
or `/path/to/file`. Replace them with one exact inspected value. For example:

```bash
systemctl status firewalld.service --no-pager
```

Do not run this literal pattern:

```text
systemctl status NAME.service --no-pager
```

Code font does not automatically mean “paste unchanged”. Read the surrounding
sentence and the command synopsis first.

### `--` ends option processing when the program supports it

A filename may begin with `-`. Without an option terminator, a program may
interpret it as an option:

```bash
printf '%s\n' 'example' > ./-notes
ls -l -- ./-notes
```

Many Unix utilities recognize `--` as the end of options. Everything after it
is then an operand even if it begins with a hyphen. This convention is common,
not a universal law; verify each program's documentation.

## How Bash finds a command

First determine what the current shell believes a name means:

```bash
type -a cd
type -a printf
type -a rg
command -V systemctl
command -v bash
```

`type` can identify aliases, functions, builtins, keywords, and executable
paths. `command -v` prints the command Bash would use, but its result is not
guaranteed to be a filesystem path: it may identify a builtin, function, or
alias. Use `type -a` when learning or diagnosing shadowed names.

At a high level:

1. Bash parses shell syntax and expands aliases where aliases are applicable.
2. For a simple command name without `/`, command resolution considers shell
   functions, builtins, and executables found through `PATH`; Bash may cache a
   prior path lookup.
3. A name containing `/` is treated as a path and is not searched through
   `PATH`.

`which` is often an external program that searches `PATH` but cannot fully
describe the current shell's aliases, functions, or builtins. Prefer Bash's
own `type` and `command` builtins for this question.

### `PATH` is an ordered search list

Inspect it without making it unreadable:

```bash
printf '%s\n' "${PATH//:/$'\n'}"
```

Each colon-separated directory is searched in order. The first matching
executable normally wins. Adding a writable or untrusted directory early in
`PATH` can cause an unexpected program to run under a familiar name.

The current directory is not implicitly searched merely because a file is
visible in `ls`. Run an executable in it with an explicit path:

```bash
./script.sh
```

This makes the trust decision visible and avoids allowing arbitrary files in
every visited directory to shadow system commands.

If a newly installed or replaced executable is not found as expected, inspect
and clear Bash's path cache rather than reopening the terminal blindly:

```bash
type -a COMMAND_NAME
hash -t COMMAND_NAME
hash -r
```

Replace `COMMAND_NAME`; `hash -t` returns an error if Bash has no cached entry.

## Paths and the current working directory

Every process has a current working directory. Inspect the shell's directory
with:

```bash
pwd
printf '%s\n' "$PWD"
```

`cd` changes Bash's own working directory, so later relative paths resolve
from the new location. An external child process cannot change the parent
shell's working directory after it exits; this is one reason `cd` is a shell
builtin.

### Absolute and relative paths

| Form | Interpretation |
| --- | --- |
| `/etc/pacman.conf` | Absolute path beginning at filesystem root |
| `docs/README.md` | Relative to the current working directory |
| `./script.sh` | Object named `script.sh` in the current directory |
| `../README.md` | `README.md` in the parent directory |
| `~/.config/niri/config.kdl` | Tilde expansion to the current user's home, then a relative suffix |
| `$HOME/.config/niri/config.kdl` | Parameter expansion of the home path |

Filesystem root `/`, the root user's home `/root`, and an administrative root
shell are three different concepts.

On this ext4 installation, names are case-sensitive: `README.md` and
`readme.md` can be different files. A leading dot makes a name conventionally
hidden from default directory listings; it does not encrypt, protect, or make
the object special to the kernel.

Filename extensions are conventions used by people and applications. Linux
does not decide whether a regular file is executable from `.exe`; it uses file
type, format or interpreter information, and execute permission.

### Inspect before resolving or modifying

```bash
ls -ld -- /etc/systemd/logind.conf.d
stat -- /etc/systemd/logind.conf.d
file -- /boot/EFI/Linux/arch-linux.efi
readlink -- /etc/resolv.conf
readlink -f -- /etc/resolv.conf
```

`ls -ld` describes the directory object rather than listing its contents.
`stat` provides metadata. `file` examines content signatures. `readlink`
shows a symbolic link's stored target; `readlink -f` canonicalizes the path
through links and `..` components when the required path components exist.

Do not parse human-oriented `ls` output in a script. Filenames can contain
spaces, tabs, quotes, glob characters, and even newlines. Use programmatic
interfaces, null-delimited records, arrays, or a language with path objects.

## Quoting and expansion

Bash does not simply split the typed line once at spaces. It parses syntax and
performs an ordered series of expansions. A practical simplified model is:

1. parse operators, quoting, and shell grammar;
2. brace expansion;
3. tilde, parameter, arithmetic, command, and process substitution;
4. word splitting;
5. pathname expansion, commonly called globbing;
6. quote removal;
7. apply redirections and execute the resulting command.

Some phases and contexts have important exceptions. Use this model to ask the
right question, then use `man bash` for exact behavior.

### Single quotes

Single quotes preserve every enclosed character literally. They are ideal for
fixed text containing `$`, spaces, glob characters, or backslashes:

```bash
printf '%s\n' '$HOME is not expanded here'
```

A literal single quote cannot appear directly inside a single-quoted string.
Close the quote, add an escaped quote, and reopen it, or choose another safe
representation:

```bash
printf '%s\n' 'Marcos'"'"'s file'
```

### Double quotes

Double quotes preserve one shell word while still allowing parameter,
arithmetic, and command substitution:

```bash
name='Shell Lab'
printf 'Directory: %s\n' "$name"
```

The durable default is to double-quote expansions that should remain one
argument:

```bash
micro "$HOME/Documents/notes with spaces.md"
```

### Unquoted expansions are a second language pass

An unquoted variable expansion can undergo word splitting and pathname
expansion. This is unsafe when the value represents one path:

```text
cp $source $destination
```

Use:

```bash
cp -- "$source" "$destination"
```

Quoting does not validate that the variables contain the intended paths; it
only preserves each value as one argument. Inspect important paths separately.

### Backslash

In Bash, an unquoted backslash preserves the literal meaning of the next
character. A backslash immediately followed by a newline removes that newline,
continuing one logical command:

```bash
printf '%s\n' \
  'first argument' \
  'second argument'
```

There must be nothing, including a trailing space, after the continuation
backslash. This syntax is for Bash, not the Windows PowerShell handoff.

### Tilde expansion

Tilde expansion occurs only in particular unquoted positions. This works:

```bash
printf '%s\n' ~/.config
```

This prints a literal tilde path because the tilde is quoted:

```bash
printf '%s\n' "~/.config"
```

Use `"$HOME/.config"` when a home path must be inside a larger quoted word.

### Brace expansion is textual generation

```bash
printf '%s\n' file-{a,b,c}.txt
printf '%s\n' chapter-{01..03}.md
```

Brace expansion does not check which files exist. It generates text before
pathname expansion and can multiply destructive operands rapidly. Expand with
`printf` first when the result matters.

### Globs are not regular expressions

Shell pathname patterns are expanded by Bash against directory entries before
the program starts:

| Glob | Typical meaning |
| --- | --- |
| `*` | Any string within one path component |
| `?` | One character |
| `[abc]` | One character from the set |
| `[0-9]` | One character from the locale-dependent range |

By default, `*` does not match leading-dot names, and an unmatched pattern may
remain literal. Those behaviors can be changed by shell options, which is why
a copied script should not silently assume an interactive configuration.

A regular expression belongs to a program or a Bash conditional such as
`[[ value =~ regex ]]`. `rg '*.conf'` does not mean “all `.conf` files”; the
pattern is an invalid or unintended regex. Use a glob-aware option or an
appropriate regex according to `rg --help`.

### Arrays preserve several independent paths

A string is not a safe substitute for a list of filenames. Use a Bash array:

```bash
files=(
  "$HOME/Documents/first note.md"
  "$HOME/Documents/second note.md"
)

printf '%s\n' "${files[@]}"
```

`"${files[@]}"` expands to one argument per element. `"${files[*]}"` joins
the elements into one word using the first character of `IFS`; that is a
different operation.

### Command substitution removes trailing newlines

```bash
kernel_release=$(uname -r)
printf 'Kernel: %s\n' "$kernel_release"
```

`$(...)` captures standard output and removes trailing newline characters. It
does not capture standard error. Do not store arbitrary binary data or a list
of filenames in a scalar command substitution.

## Variables, environment, and process inheritance

A Bash shell variable exists inside the current shell:

```bash
project_name='arch-linux-handbook'
printf '%s\n' "$project_name"
```

An exported variable is placed in the environment inherited by subsequently
started child processes:

```bash
export project_name
env | rg '^project_name='
```

The child receives a copy of the environment. It cannot retroactively change
the parent shell's variable. A temporary assignment can affect one command:

```bash
LC_ALL=C sort --version
```

That does not permanently change the shell's locale variable.

Inspect environment and shell state with distinct tools:

```bash
printenv HOME
printenv PATH
declare -p HOME PATH
set -o
shopt
```

`printenv` sees exported environment variables. `declare -p` asks Bash about
its variables and attributes. `set -o` and `shopt` describe two families of
Bash options.

Do not place every variable in `/etc/environment` or `.bashrc`. Environment
scope is architecture:

- a shell variable affects the current shell;
- an exported variable reaches later child processes;
- a graphical application launched before the export does not travel back in
  time to receive it;
- the systemd user manager has its own activation environment;
- system services have unit-specific environments;
- locale, XDG, application, session, and secret values each have more suitable
  owners.

Guides 03, 14, and 18 explain systemd user scope, Niri session environment,
and XDG boundaries.

### Do not put secrets in ordinary command lines

Command arguments may appear in shell history, terminal scrollback, process
inspection, logs, screenshots, and support transcripts. Environment variables
are also not a general secret store. Prefer a program's protected prompt,
file-descriptor, credential, keyring, or documented secret mechanism.

If a secret was entered, rotating it is usually more reliable than assuming a
history deletion erased every copy. Other shells, terminals, logs, clipboard
managers, process monitors, or synchronized history may already have observed
it.

## Standard streams and file descriptors

Most terminal programs begin with three open file descriptors:

| Descriptor | Name | Default interactive destination |
| --- | --- | --- |
| `0` | standard input, stdin | Keyboard input through the terminal |
| `1` | standard output, stdout | Normal output to the terminal |
| `2` | standard error, stderr | Diagnostics to the terminal |

The distinction is semantic, not cosmetic. A program can send machine-readable
results to stdout and warnings to stderr so that the results can be piped or
saved without mixing both streams.

### Write, append, and read redirection

```bash
printf '%s\n' 'first line' > notes.txt
printf '%s\n' 'second line' >> notes.txt
wc -l < notes.txt
```

- `>` opens the target for writing and normally truncates an existing file;
- `>>` opens it for append;
- `<` makes a file the command's stdin.

The shell opens the redirection before starting the external program. A typo
can therefore empty a file even when the program later fails. Inspect the
target, prefer unique output names, and use an editor or a validated temporary
file when the content matters.

### Redirect stderr separately

```bash
some-command > output.log 2> error.log
some-command >> output.log 2>> error.log
```

These are structural examples; replace `some-command` with a known safe
command. File descriptor `2` names stderr. Omitting the number before `>`
means stdout, descriptor `1`.

### Redirection order matters

Bash processes redirections from left to right:

```bash
some-command > combined.log 2>&1
```

First stdout points to `combined.log`; then stderr is duplicated to wherever
stdout points now. Both streams enter the file.

```bash
some-command 2>&1 > output-only.log
```

First stderr is duplicated to stdout's original terminal destination; then
only stdout moves to `output-only.log`. Stderr still appears on the terminal.

`2>&1` does not mean “send file 2 to a file named 1”. It duplicates one open
file descriptor onto another. Preserve the order shown in reviewed commands.

### Pipes connect stdout to stdin

```bash
printf '%s\n' beta alpha gamma | sort
```

Bash connects the stdout of the left command to the stdin of the right
command. Stderr is not part of a normal `|` pipeline and remains visible.
Bash also supports `|&` as a shorthand for piping stdout and stderr together,
but combining diagnostic text with data can make downstream parsing invalid.

A pipeline streams data. It does not necessarily wait for the left command to
finish and store all output first. Backpressure, buffering, early exits, and
signals can affect each process independently.

### `tee` both displays and writes

```bash
printf '%s\n' alpha beta | tee -- output.txt
```

`tee` copies stdin to stdout and named files. Use `tee -a` to append. Without
`-a`, it can truncate an existing target just like `>`.

The common `sudo` redirection trap is:

```text
sudo printf '%s\n' value > /etc/example.conf
```

Bash performs `>` as the current unprivileged user before `sudo` starts
`printf`, so this normally fails to open the protected target. Do not solve
the problem by opening a root shell casually. For an existing configuration
file, prefer a reviewed editor flow:

```bash
SUDO_EDITOR=micro sudoedit /etc/example.conf
```

`sudoedit` creates a temporary user-editable copy, runs the editor as the
invoking user, and copies a modified result back under policy control. For
generated content, a reviewed `sudo tee` pipeline can be appropriate, but it
still overwrites or appends exactly as requested and must not be treated as a
universal substitute for an editor.

Special configuration such as `/etc/sudoers` must use its dedicated validator
(`visudo`), as guide 02 explains.

### `/dev/null` belongs to Unix-like systems

On Arch, `/dev/null` accepts and discards writes:

```bash
some-command > /dev/null
some-command > /dev/null 2>&1
```

The first discards only stdout; the second discards stdout and then stderr.
The command still runs, can change state, and returns an exit status. Silence
is not safety.

`/dev/null` is not a Windows path. In PowerShell, redirect only the intended
PowerShell stream to `$null`, for example `*> $null` for all redirectable
streams, or use `Out-Null` according to the command's semantics. PowerShell
has more streams than Bash and its pipeline normally carries objects, so this
is not a character-for-character translation.

### Use `less` for long output instead of destroying it

```bash
systemctl list-unit-files --no-pager | less
```

Useful `less` controls include:

| Key | Action |
| --- | --- |
| `Space` or `PageDown` | Next screen |
| `b` or `PageUp` | Previous screen |
| `/text` | Search forward |
| `n` / `N` | Next / previous match |
| `g` / `G` | Start / end |
| `q` | Quit |

Many commands automatically use a pager only when attached to a terminal.
`--no-pager` is useful in evidence capture or another explicit pipeline; it is
not always more readable interactively.

## Exit status and conditional execution

Every completed simple command has an integer exit status. By Unix convention,
zero means success and nonzero means some other result. The meaning of a
particular nonzero value belongs to that program.

Inspect Bash's most recent status immediately:

```bash
test -d /etc
printf 'status=%s\n' "$?"
```

Running `printf` replaces `$?`, so a later inspection would describe
`printf`, not `test`.

No terminal output does not imply success, and visible output does not imply
failure. For example, a quiet validation can succeed; a search tool can use a
nonzero status to mean “no matches” rather than an internal crash. Read the
program's `EXIT STATUS` section.

### `&&`, `||`, `;`, and newline

```bash
first-command && second-command
first-command || recovery-command
first-command; second-command
```

- `&&` runs the right side only if the left side returns zero;
- `||` runs the right side only if the left side returns nonzero;
- `;` and a newline sequence commands without making the second conditional
  on the first command's success.

These operators are control flow, not decorative separators. Avoid long
one-liners that hide which step failed. This handbook normally keeps risky
commands on separate lines so their output can be read before continuing.

### Pipeline status needs special attention

By default, a Bash pipeline's status is the status of its final command:

```bash
producer | consumer
```

If `producer` fails but `consumer` successfully processes no input, the
pipeline may still report zero. Bash provides:

```bash
set -o pipefail
```

With `pipefail`, the pipeline is nonzero when a component fails, using the
rightmost nonzero status. Bash also exposes the most recent pipeline's
component statuses in `PIPESTATUS`:

```bash
printf '%s\n' beta alpha | sort
pipeline_status=("${PIPESTATUS[@]}")
printf 'component statuses: %s\n' "${pipeline_status[*]}"
```

Do not enable `pipefail`, `errexit`, or a copied “strict mode” globally in the
interactive shell without understanding the scripts and startup files it
affects.

### `set -euo pipefail` is not a correctness proof

This popular line combines several Bash behaviors:

- `-e`: exit in many, but not all, contexts after a nonzero status;
- `-u`: treat some unset-parameter expansions as errors;
- `pipefail`: expose failures hidden before the final pipeline component.

Each has exceptions and can change intended control flow. It does not validate
paths, prevent injection, make temporary files safe, make cleanup reliable, or
prove that a successful command produced the desired state. Use explicit
checks and test each script's failure paths.

## Foreground jobs, signals, and terminal lifetime

An interactive Bash with job control gives the terminal foreground to one job.
Common controls are:

| Input or command | Meaning |
| --- | --- |
| `Ctrl+C` | Terminal normally sends `SIGINT` to the foreground process group |
| `Ctrl+Z` | Terminal normally sends `SIGTSTP`, stopping the foreground job |
| `jobs -l` | List jobs and process IDs known to this Bash |
| `fg %1` | Continue job 1 in the foreground |
| `bg %1` | Continue stopped job 1 in the background |
| `command &` | Start an asynchronous background job |
| `kill PID` | Request termination with `SIGTERM` by default |
| `kill -KILL PID` | Force kernel termination; last resort with no cleanup opportunity |

Applications may handle or ignore signals. `Ctrl+C` is not an unconditional
undo operation, and it cannot reverse changes already written before the
signal arrived.

When an interactive shell exits, terminal and shell lifetime can affect its
jobs, commonly through `SIGHUP`. `nohup`, `disown`, terminal multiplexers, and
backgrounding have different semantics and failure reporting. A program that
must survive logout, restart, report logs, or obey session dependencies should
normally become a reviewed systemd user or system unit, not an unexplained
`&` in `.bashrc`.

## Safe file and directory operations

### Inspect the operands first

Before a material copy, move, sync, or removal:

```bash
pwd
printf 'source=%q\n' "$source"
printf 'destination=%q\n' "$destination"
ls -ld -- "$source" "$destination"
```

`%q` prints a Bash-reusable representation that makes spaces and special
characters more visible. It does not prove the path is correct.

### Create directories deliberately

```bash
mkdir -- "$HOME/Documents/new-directory"
mkdir -p -- "$HOME/Documents/project/docs"
```

Without `-p`, an existing target is an error. With `-p`, existing parent
directories are accepted and missing components are created. A misspelled
absolute path can therefore create an unintended tree; inspect it first.

### Copy with explicit source and destination

```bash
cp -- source.txt destination.txt
cp -a -- source-directory destination-parent
```

`cp -a` preserves more metadata and copies directories recursively. It is not
always the desired semantics for files crossing filesystems or trust
boundaries. Check whether the destination exists: copying into an existing
directory and creating a new directory path are different operations.

For installing a reviewed file with exact mode and parent creation, `install`
can make the policy explicit:

```bash
sudo install -Dm644 -- reviewed.conf /etc/example/example.conf
```

This is a structural example, not authorization to create that fictional
configuration. Validate the real program's path, owner, mode, syntax, reload,
and rollback requirements first.

### Move and rename

```bash
mv -i -- old-name new-name
```

`-i` asks before overwriting an existing destination, but interactive prompts
are unsuitable as the only safety mechanism in automation. A move across
filesystems can become copy plus removal and is not necessarily atomic.

### `rm` is not a trash operation

`rm` removes directory entries directly and has no general undo. Avoid broad
recursive examples, unquoted variables, and globs whose expansion has not been
printed and reviewed. For ordinary desktop files, use Nautilus and its Trash
workflow when recoverability is useful.

If a filename begins with `-`, use an option terminator and explicit path:

```bash
rm -- ./-notes
```

This exact command removes the lab file created earlier; do not generalize it
to unidentified paths. `rm -rf` is not a cleanup shortcut and does not belong
in a learning exercise.

### `rsync` trailing slashes change meaning

First inspect a dry run:

```bash
rsync -a --dry-run --itemize-changes -- source-directory/ destination-directory/
```

For rsync, `source-directory/` means the contents of the source directory,
whereas `source-directory` commonly creates or addresses that directory under
the destination. Preserve or change the trailing slash deliberately. Do not
add `--delete` until the exact source and destination are proven and a backup
exists; it authorizes destination removal.

## Finding files and searching content

Choose the tool according to the question:

| Question | Tool and example |
| --- | --- |
| What is directly in this directory? | `ls -la -- .` |
| What is the visible tree structure? | `tree -a -L 2 -- .` |
| Which path names match quickly? | `fd --hidden --type f 'README' .` |
| Which filesystem objects satisfy precise predicates? | `find . -type f -name '*.md' -print` |
| Which text files contain a regex? | `rg --line-number 'pattern' .` |
| Which text files contain literal punctuation? | `rg --line-number --fixed-strings 'literal[text]' .` |
| Which package owns an installed path? | `pacman -Qo /absolute/path` |
| Which files does an installed package own? | `pacman -Ql PACKAGE` |

Quote a glob intended for `find` so Bash does not expand it prematurely:

```bash
find . -type f -name '*.md' -print
```

`rg` respects ignore files by default and skips hidden and binary content in
common cases. Options such as `--hidden`, `--no-ignore`, and `--binary` widen
the search and can expose caches, repositories, secrets, or large data. Widen
scope intentionally.

### Filenames require robust delimiters

Newline-delimited output is convenient for humans but cannot represent every
Unix filename unambiguously. When one program produces paths for another,
prefer a direct `-exec` action or null delimiters:

```bash
find . -type f -name '*.md' -print0 | xargs -0 --no-run-if-empty wc -l
```

This lab-scale command handles spaces and newlines. Before substituting a
modifying command for `wc`, inspect the path set and understand `find` and
`xargs` concurrency, batching, and race boundaries.

Never use `for file in $(find ...)` for arbitrary filenames; command
substitution, word splitting, and globbing corrupt the path boundaries.

## Text viewing, filtering, and structured data

Useful read-only building blocks include:

```bash
head -n 20 -- file.txt
tail -n 20 -- file.txt
tail -f -- application.log
sort -- file.txt
sort -u -- file.txt
wc -l -- file.txt
rg --line-number --fixed-strings 'needle' file.txt
bat --paging=always -- file.txt
jq . document.json
```

Stop `tail -f` with `Ctrl+C`. `jq .` parses JSON and formats it; an error is
evidence that the input is incomplete or invalid JSON. Do not use regular
expressions as a general parser for JSON, KDL, unit files, or shell syntax.

Pipelines should keep one representation at each boundary. Human-formatted
tables can truncate, align, color, localize, or escape values and are poor
machine interfaces. Prefer a program's JSON, null-delimited, raw, or explicit
field output when available.

## A safe shell laboratory

Practice below the unprivileged `Documents` directory, not in `/etc`, `/boot`,
or the repositories. `mktemp` creates a unique private directory and prints
its exact path:

```bash
lab_dir=$(mktemp -d --tmpdir="$HOME/Documents" shell-lab.XXXXXX)
printf 'Lab: %s\n' "$lab_dir"
cd -- "$lab_dir"
mkdir -- 'input data' output
printf '%s\n' alpha beta gamma > 'input data/names.txt'
printf '%s\n' 'beta note' 'delta note' > 'input data/notes with spaces.txt'
pwd
find . -maxdepth 2 -type f -print
```

Now observe one stream at a time:

```bash
rg --line-number --fixed-strings beta 'input data'
rg --line-number --fixed-strings beta 'input data' > output/matches.txt
wc -l < output/matches.txt
cat -- output/matches.txt
```

Compare redirection order with a command that emits both streams:

```bash
bash -c 'printf "normal\n"; printf "diagnostic\n" >&2' > output/combined.txt 2>&1
cat -- output/combined.txt

bash -c 'printf "normal\n"; printf "diagnostic\n" >&2' 2>&1 > output/normal-only.txt
cat -- output/normal-only.txt
```

In the second case, `diagnostic` appears on the terminal while the file holds
only `normal`.

Test quoting and arrays:

```bash
files=('input data/names.txt' 'input data/notes with spaces.txt')
printf '<%s>\n' "${files[@]}"
wc -l -- "${files[@]}"
```

Open the lab in Nautilus and move it to Trash when finished:

```bash
nautilus -- "$lab_dir"
```

Do not turn the variable into a recursive `rm` example. The deliberate Trash
step teaches recovery and gives one last visual check of the unique target.

## Editors and privileged editing

### Micro is the canonical terminal editor

Open one or more files as the ordinary user:

```bash
micro "$HOME/Documents/notes.md"
```

The essential default keys are:

| Key | Action |
| --- | --- |
| `Ctrl+S` | Save |
| `Ctrl+Q` | Quit |
| `Ctrl+G` | Open Micro's in-editor help |

Micro's configuration resolves through `MICRO_CONFIG_HOME`, then
`XDG_CONFIG_HOME/micro`, then `~/.config/micro`. The project has not created a
Micro configuration package yet; do not assume plugin, colorscheme, or
keybinding changes are portable between the two ThinkPads.

### Prefer `sudoedit` over running the editor as root

For an ordinary root-owned configuration that has no mandatory specialized
editor:

```bash
SUDO_EDITOR=micro sudoedit /etc/example.conf
```

The assignment selects Micro for that one invocation. `sudoedit` edits a
temporary copy as the invoking user and installs the changed copy only after
the editor exits. This reduces the amount of editor code running with elevated
privilege and keeps user editor state out of root's home.

It does not validate the application's syntax, reload its service, or create a
rollback automatically. The correct workflow remains:

1. inspect the existing file and its owner;
2. identify package versus administrator ownership;
3. preserve or version the previous policy;
4. edit the documented source file;
5. run the program-specific validator;
6. inspect the merged/effective configuration;
7. reload or restart only when required;
8. verify behavior and retain a recovery route.

Guide 01 explains drop-ins and precedence. Guide 02 explains `visudo`, PAM,
permissions, and privilege boundaries. Guide 03 explains service reload and
restart semantics.

### Vim is a recovery literacy skill, not the current editor choice

The project installs Vim without plugins or user configuration, but it does
not select Vim as the canonical editor. Micro remains the convenient default
for terminal editing. A recovery environment, remote host, or dedicated
command such as `visudo` may still present a vi-like editor. The smallest
survival model is:

| Input | Meaning in a vi-compatible editor |
| --- | --- |
| `i` | Enter insert mode |
| `Esc` | Return to normal mode |
| `:w` then Enter | Write the file |
| `:q` then Enter | Quit if no unsaved change blocks it |
| `:wq` then Enter | Write and quit |
| `:q!` then Enter | Quit and discard unsaved editor-buffer changes |

These commands do not apply to Micro. Check the editor actually on screen
before using them. A future editor-comparison guide or dotfile decision may go
deeper; the installed Vim remains an uncustomized secondary and recovery tool.

### Editor-selection variables

Programs commonly consult `SUDO_EDITOR`, `VISUAL`, and `EDITOR`, but order and
scope are program-specific. `sudoedit` under the sudoers policy checks
`SUDO_EDITOR`, then `VISUAL`, then `EDITOR`. Other programs may use only one.

A one-command assignment avoids making an unreviewed session-wide policy:

```bash
EDITOR=micro some-command
```

Replace `some-command` only after reading its manual. When the permanent shell
environment is designed later, the project can decide whether `EDITOR` and
`VISUAL` belong in a portable Bash package or a wider session environment.

## Bash startup files and configuration scope

Bash reads different files depending on whether it is interactive, a login
shell, invoked as `sh`, or running a non-interactive script.

Upstream Bash's principal user-file rules are:

| Invocation | User startup behavior |
| --- | --- |
| Interactive login shell | Reads the first readable file among `~/.bash_profile`, `~/.bash_login`, and `~/.profile` after `/etc/profile` |
| Interactive non-login shell | Reads `~/.bashrc` |
| Login shell exiting | Reads `~/.bash_logout` if present |
| Non-interactive Bash script | Does not normally read `~/.bashrc`; may consult `BASH_ENV` |
| Invoked as `sh` | Uses different, more POSIX-oriented startup behavior |

Arch also supplies `/etc/bash.bashrc` through its Bash packaging and its
default `/etc/profile`/skeleton files connect the system and user startup
paths. The installed `bash-completion` framework is sourced through Arch's
system Bash configuration; it does not require copying a large completion
block into the user's `.bashrc`.

Inspect actual files and invocation instead of guessing:

```bash
printf 'shell=%s\n' "$SHELL"
printf 'argv0=%s\n' "$0"
printf 'flags=%s\n' "$-"
shopt -q login_shell; printf 'login_shell status=%s\n' "$?"
type -a _completion_loader
ls -la -- "$HOME"/.bash*
```

The presence of `i` in `$-` indicates an interactive Bash. The `shopt` status
must be read immediately: zero means the `login_shell` option is set.

### `.bashrc` is not a session autostart directory

Every new interactive Bash may read `.bashrc`. Starting udiskie, Mako,
Waybar, a polkit agent, an SSH agent, or another long-lived desktop process
there can create duplicates for every terminal. This project uses Niri spawn
rules, XDG autostart, D-Bus activation, or systemd user units according to the
component's lifecycle.

Likewise, shell-only exports typed in Kitty do not automatically repair the
already-created Niri, portal, or systemd user environment. Guide 14 explains
the graphical-session boundary.

### Do not source untrusted files

These execute a file inside the current Bash process:

```bash
source ./environment.sh
. ./environment.sh
```

They are not imports of inert key-value data. The sourced file can change
variables, options, functions, aliases, working directory, traps, file
descriptors, and even exit the current shell. Read and trust it first.

## Shell scripts: execution is different from sourcing

Create a small script only inside the lab:

```bash
micro "$lab_dir/report.sh"
```

Example content:

```bash
#!/usr/bin/bash

printf 'user=%s\n' "$USER"
printf 'directory=%s\n' "$PWD"
```

Validate syntax without executing commands:

```bash
bash -n -- "$lab_dir/report.sh"
```

Run it explicitly through Bash:

```bash
bash -- "$lab_dir/report.sh"
```

Or make it executable and let the kernel use the shebang:

```bash
chmod u+x -- "$lab_dir/report.sh"
"$lab_dir/report.sh"
```

The shebang is the first line naming the interpreter. `/usr/bin/bash` is the
known Bash path on this Arch project. `#!/usr/bin/env bash` searches `PATH` and
can be useful for source portability, but it also delegates interpreter
selection to that environment. Choose deliberately.

Running a script creates a separate Bash execution environment; ordinary
variable and directory changes do not return to the parent. Sourcing the same
file executes it inside the current shell and is a much larger trust and state
decision.

`bash -n` checks Bash syntax only. It does not prove paths, permissions,
dependencies, quotations under every input, idempotence, rollback, or safe
failure. ShellCheck can add static analysis later, but this guide installs no
new package and no linter replaces testing.

## History and interactive editing

Bash normally uses Readline for interactive command-line editing and history.
Useful default concepts include:

- Up/Down to recall nearby history entries;
- `Ctrl+R` for reverse incremental search;
- `Ctrl+A` and `Ctrl+E` for beginning and end of line;
- `Ctrl+U` and `Ctrl+K` for deleting around the cursor;
- Tab for completion, extended by `bash-completion` for supported commands.

Before pressing Enter on recalled history, reread the entire line. A previous
path, branch, package, device, or redirect target may no longer be correct.

Bash interactive history expansion uses `!` by default. It happens before
normal word parsing and can surprise pasted text containing exclamation marks.
Use single quotes for literal fixed text, inspect an expanded line, or disable
the feature only through a deliberate shell policy. Do not place passwords or
tokens in history merely because deletion commands exist.

## Reading command documentation

### Start by identifying the command

```bash
type -a cd
type -a printf
type -a systemctl
```

Then choose the right documentation source:

| Object | First documentation step |
| --- | --- |
| Bash builtin such as `cd` | `help cd` or `help -m cd` |
| External command | `command --help`, then its manual page |
| Configuration file | Manual section 5, package docs, and effective-config tools |
| System administration command | Often manual section 8 |
| GNU utility with extensive detail | `info coreutils`, then the relevant node |
| Arch integration or supported procedure | Current ArchWiki and Arch News |
| Project-specific decision | This handbook plus runbook/post-install source |

`help` is a Bash builtin. `man cd` may show a POSIX page or another document,
while `help cd` describes the Bash builtin actually changing this shell.

### Read the synopsis as a grammar

Manual pages conventionally use:

| Notation | Meaning |
| --- | --- |
| Bold or literal command text | Type exactly as shown |
| Italic or underlined argument | Replace with a suitable value |
| `[OPTION]` | Optional element |
| `A\|B` | Alternatives, commonly mutually exclusive in that position |
| `ARGUMENT...` | Repeatable argument |
| `NAME(5)` | Manual page named `NAME` in section 5 |

The square brackets and ellipsis are normally notation, not characters to
type. A synopsis describes accepted shapes; it does not tell which shape is
safe for the current system.

### Manual sections prevent name collisions

```bash
man man
man 1 printf
man 3 printf
man 5 passwd
man 8 sudo
```

The major sections are:

| Section | Content |
| --- | --- |
| 1 | User commands and shell commands |
| 2 | Kernel system calls |
| 3 | Library calls |
| 4 | Special files and devices |
| 5 | File formats and conventions |
| 6 | Games |
| 7 | Miscellaneous conventions and overviews |
| 8 | System-administration commands |
| 9 | Kernel routines on systems that provide them |

Search by description when the exact page is unknown:

```bash
apropos 'copy files'
man -k 'network manager'
whatis bash
```

`apropos` uses a manual-page index. A newly installed page may not appear
immediately until the supplied man-db update mechanism refreshes that index.

### Navigate a manual page

`man` normally opens `less`, so the earlier pager keys apply. Useful searches
are:

```text
/^SYNOPSIS
/^OPTIONS
/EXIT STATUS
/FILES
/ENVIRONMENT
/EXAMPLES
/SEE ALSO
```

Not every page has every section. Read `EXIT STATUS`, `FILES`, and
`ENVIRONMENT`; many command failures and configuration surprises are explained
there rather than in the first example.

### GNU Info complements concise manual pages

GNU projects often keep their complete manual in Info:

```bash
info bash
info coreutils
info '(coreutils) cp invocation'
```

Press `h` inside Info for its navigation help and `q` to quit. The man page is
usually faster for an option; Info is often better for concepts, examples,
and linked chapters.

### Ask the package database what is installed

```bash
pacman -Q bash
pacman -Qi bash
pacman -Qo /usr/bin/bash
pacman -Ql bash | less
```

These answer different questions: installed version, package metadata, owner
of an exact path, and files owned by the installed package. A package file list
does not include every runtime-generated cache or user data file.

Use `pacman -Qo` on an existing exact path. Use guide 04 before refreshing
file databases, installing documentation, or changing package state.

### Choose sources by authority and layer

A practical order is:

1. local `--help`, `help`, and manual/Info pages matching the installed
   version;
2. package-owned examples and `/usr/share/doc` where provided;
3. current upstream documentation for program behavior;
4. ArchWiki for Arch packaging, integration, procedures, and warnings;
5. this project for the choices made for these two ThinkPads;
6. forums, issue trackers, and third-party guides as evidence tied to exact
   versions, not timeless instructions.

The provided offline ArchWiki copy is valuable during recovery and for stable
concepts, but it is a dated snapshot. Before a package, boot, security,
firmware, or migration decision, compare with current primary documentation
when connectivity exists.

Record versions with evidence:

```bash
bash --version
pacman -Q bash
uname -r
```

Do not paste a command from an answer whose distribution, shell, init system,
boot manager, filesystem, or version assumptions differ from this project.

## Reading this project's command blocks safely

Before executing a block, answer:

1. Which machine: Windows handoff, installed Arch, TTY, Niri terminal, or Arch
   ISO?
2. Which interpreter: PowerShell, Bash, another shell, or no interpreter
   because the block is only text?
3. Which user and privilege boundary?
4. Which directory and mounted filesystem?
5. Which words are placeholders?
6. Which commands only inspect and which mutate state?
7. Where do stdout and stderr go?
8. What exit status or visible state proves success?
9. What is the rollback or recovery path?
10. Does the command still match the current project profile?

Commands in one fenced block are not automatically one atomic transaction.
Stop after unexpected output. Do not continue merely because more lines remain
below it.

For a risky command, decompose it:

- print variable values with `%q`;
- replace a mutating action with a list or dry-run mode where the program
  genuinely supports one;
- list glob expansion with `printf`;
- inspect source and destination separately;
- capture the relevant current state;
- run one change;
- validate the effective result.

A `--dry-run` option is program-specific and may still perform discovery,
authentication, network, locking, or partial setup. Read its manual rather
than assuming the name promises zero side effects.

## Troubleshooting by layer

| Symptom | Likely boundary | Inspect first |
| --- | --- | --- |
| `command not found` | Bash resolution or package state | `type -a NAME`, `printf '%s\n' "$PATH"`, `pacman -Q` |
| File is visible but `name: command not found` | Current directory is not in `PATH` | Verify trust and execute `./name` if appropriate |
| `Permission denied` executing a file | Mode, mount option, interpreter, or directory traversal | `namei -l`, `stat`, `findmnt`, shebang; see guide 02 |
| `No such file or directory` for an existing script | Missing shebang interpreter, CRLF, broken link, or wrong working directory | `pwd`, `file`, `head -n 1`, `readlink -f` |
| Path with spaces becomes several errors | Missing quoting or unsafe word splitting | `printf '%q\n' "$path"`; quote the expansion |
| `*` is passed literally | No match or shell-option difference | `printf '<%s>\n' pattern`, `shopt nullglob failglob dotglob` |
| Output file becomes empty | `>` truncated it before command success | Stop writes; inspect backups/history and command order |
| Error still appears after `>file` | Only stdout was redirected | Preserve diagnostics or redirect `2>` deliberately |
| Pipeline reports success despite earlier failure | Final-command pipeline status | Capture `PIPESTATUS`; consider reviewed `pipefail` |
| `sudo echo ... > /etc/file` fails | Shell opened redirect before sudo | Use `sudoedit` or a reviewed `sudo tee` design |
| `.bashrc` change does not affect Niri | Wrong lifecycle and environment owner | Guide 14 session environment; systemd user environment |
| Component starts once per terminal | Long-lived process incorrectly placed in `.bashrc` | Move lifecycle to Niri, XDG autostart, D-Bus, or systemd user scope |
| Bash multiline command fails in PowerShell | Wrong language continuation/parser | Use the package's one-line PowerShell command |
| PowerShell reports `\` outside the repository | Bash continuation pasted into PowerShell | Remove `\`; run the complete Git command on one line |
| `/dev/null` fails on Windows | Unix device path used in PowerShell | Use the appropriate PowerShell stream and `$null` syntax |
| Manual search finds nothing | Wrong page/section or stale apropos index | `type -a`, `man -a`, exact section, package ownership |
| Editor appears “stuck” | Unknown editor mode or foreground job | Identify Micro versus vi; use its documented quit/help keys |

### `Permission denied` does not always mean “add sudo”

It may mean:

- a regular file lacks execute permission;
- a parent directory lacks search permission;
- the filesystem is mounted `noexec`;
- a script's interpreter is inaccessible;
- the current user should not modify that object;
- the program deliberately dropped privilege;
- a mandatory security or sandbox policy denied access.

Adding `sudo` can change `$HOME`, environment, ownership of new files, PATH,
configuration source, D-Bus session access, and the damage radius. Diagnose
the denied operation and intended owner first.

### “No such file” can describe the interpreter

If an executable script visibly exists but execution says it does not, inspect:

```bash
file -- ./script.sh
head -n 1 -- ./script.sh
readlink -f -- ./script.sh
```

A shebang naming a missing interpreter, Windows CRLF characters after the
interpreter path, or a broken symbolic link can cause a misleading-looking
path error.

## Alternatives and future improvements

The current conservative baseline does not claim Bash, Kitty, or Micro is the
only good choice.

| Alternative | Potential benefit | Cost or boundary to evaluate first |
| --- | --- | --- |
| Zsh | Extensive interactive customization | Different startup/configuration ecosystem and portability assumptions |
| Fish | Friendly completion and highlighting | Deliberately non-POSIX interactive language |
| Nushell | Structured-data pipelines | Different command and scripting model |
| Starship or another prompt | Rich contextual prompt | Startup time, trust, configuration, and visual complexity |
| `fzf` shell integration | Interactive history/path selection | Keybinding ownership and sourced shell code |
| Foot | Small Wayland-native terminal | Feature/config comparison with current Kitty workflow |
| Vim/Neovim | Powerful modal editing and extensibility | Learning curve, plugin/config lifecycle, recovery simplicity |
| ShellCheck | Static shell analysis | Package and CI policy; does not prove runtime safety |

Any later Bash package should be small, reviewable, Stow-managed, and separate
portable policy from host secrets and generated history. It should first
define startup-file ownership, prompt behavior, editor variables, completion,
history privacy, `fzf` integration, aliases, script linting, and rollback.

## Decisions recorded for this project

- Kitty remains the terminal emulator and Bash remains the login and
  interactive shell.
- Micro remains the canonical terminal editor; Vim remains installed without
  user configuration as a secondary learning and recovery tool.
- PowerShell is used for Windows repository handoffs, and package Git commands
  remain on one line where practical.
- Bash and PowerShell code fences are never assumed interchangeable.
- The prompt marker is not part of a command.
- Placeholders must be replaced with exact inspected values.
- `type` and `command` are preferred over `which` for Bash command resolution.
- The current directory is executed through an explicit path such as `./name`,
  not added implicitly to `PATH`.
- Variable expansions representing one path are double-quoted.
- Arrays preserve several paths; command substitution and newline splitting do
  not form a safe filename list.
- Redirection is evaluated left to right; `>` can truncate before program
  execution.
- `/dev/null` is an Arch device path, not a portable Windows spelling.
- `sudoedit` is preferred for ordinary privileged text editing; specialized
  validators such as `visudo` retain ownership of their formats.
- `.bashrc` does not own graphical-session autostart or systemd user services.
- No aliases, prompt framework, shell plugins, editor plugins, or Bash dotfiles
  are added by this guide.
- Local documentation matching the installed version is checked before
  generic web examples.
- The offline ArchWiki copy is recovery material and a dated reference, not an
  automatic substitute for current upstream and Arch documentation.
- Destructive examples are excluded from the practice lab; ordinary cleanup
  uses the desktop Trash after visual target confirmation.

## Learning and verification checklist

- [ ] I can explain the difference between Kitty, a TTY, Bash, and the prompt.
- [ ] I know whether a command block belongs to Bash, PowerShell, or plain text.
- [ ] I do not type documentation prompt characters or literal placeholders.
- [ ] I can use `type -a` and explain why `command -v` may not print a path.
- [ ] I can distinguish absolute, relative, home-relative, and resolved paths.
- [ ] I quote a variable that represents one path and use an array for several
      paths.
- [ ] I understand the difference between a shell glob and a regular
      expression.
- [ ] I can distinguish a shell variable from an exported environment
      variable.
- [ ] I can identify stdin, stdout, stderr, and their descriptor numbers.
- [ ] I can predict the difference between `>file 2>&1` and `2>&1 >file`.
- [ ] I know that `>` may truncate a target even when the command fails later.
- [ ] I understand that a pipeline normally reports its final component's
      status and can inspect `PIPESTATUS` immediately.
- [ ] I know what `Ctrl+C`, `Ctrl+Z`, `jobs`, `fg`, and `bg` generally do.
- [ ] I inspect source, destination, expansion, and current directory before a
      material file operation.
- [ ] I know why `find ... -print0 | xargs -0` protects filename boundaries.
- [ ] I can save, quit, and open help in Micro.
- [ ] I use `sudoedit` for suitable root-owned text files and the dedicated
      validator for formats such as sudoers.
- [ ] I can identify the Bash startup file appropriate to an invocation.
- [ ] I do not start graphical-session daemons from `.bashrc`.
- [ ] I can read a manual synopsis, select a manual section, use `apropos`, and
      find package ownership.
- [ ] I practiced in a unique unprivileged lab and removed it through Trash.

## Sources

- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/)
- [`bash(1)` on Arch manual pages](https://man.archlinux.org/man/bash.1.en)
- [ArchWiki: Bash](https://wiki.archlinux.org/title/Bash)
- [ArchWiki: Command-line shell](https://wiki.archlinux.org/title/Command-line_shell)
- [ArchWiki: Core utilities](https://wiki.archlinux.org/title/Core_utilities)
- [ArchWiki: Environment variables](https://wiki.archlinux.org/title/Environment_variables)
- [ArchWiki: man page](https://wiki.archlinux.org/title/Man_page)
- [`man(1)`](https://man.archlinux.org/man/man.1.en)
- [`intro(1)`](https://man.archlinux.org/man/intro.1.en)
- [`mktemp(1)`](https://man.archlinux.org/man/mktemp.1.en)
- [`xargs(1)`](https://man.archlinux.org/man/xargs.1.en)
- [`sudo(8)`](https://man.archlinux.org/man/sudo.8.en)
- [ArchWiki: sudo](https://wiki.archlinux.org/title/Sudo)
- [`micro(1)`](https://man.archlinux.org/man/micro.1.en)
- [ArchWiki: micro](https://wiki.archlinux.org/title/Micro)
- [Kitty documentation](https://sw.kovidgoyal.net/kitty/)
- [Microsoft: PowerShell parsing](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_parsing)
- [Microsoft: PowerShell special characters](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_special_characters)
- [Microsoft: PowerShell redirection](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_redirection)

## Next guide

Guide 22 will apply this command-line foundation to Git and GitHub. It will
explain repositories, working trees, staging, commits, branches, remotes,
fetch/pull/push, identity and privacy, SSH host authentication and user keys,
GitHub CLI boundaries, Conventional Commits, daily VS Code/PowerShell and Arch
workflows, divergence, conflict recovery, and safe repair without losing local
work.
