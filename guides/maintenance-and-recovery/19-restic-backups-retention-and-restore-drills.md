# Restic backups, retention, restore drills, and recovery media

## Purpose and scope

A backup is useful only when it covers the intended data, survives the failure
being considered, and can be restored with credentials and tools that remain
available after that failure. A successful command and an encrypted repository
are important, but neither alone proves recoverability.

This guide explains the complete recovery design behind post-install chapter
12:

- what Restic protects and what it deliberately does not protect;
- how repositories, snapshots, deduplication, encryption, and locks relate;
- how source boundaries and exclusions affect the real backup;
- how to inspect, check, retain, copy, and restore snapshots safely;
- why the recovery bundle, password, Git remotes, and Arch ISO remain separate;
- which conditions must be solved before a systemd timer is appropriate;
- how the pieces are used after deletion, repository trouble, or disk failure.

It does not create a repository, select an off-site provider, activate a
retention policy, install a timer, rotate credentials, restore a LUKS header,
or repair a damaged repository. Those are operational or destructive actions
that require the live system, verified media, and a reviewed plan.

## Current project contract

The published workstation currently has this manual baseline:

| Area | Current decision |
| --- | --- |
| Restic source | `/home/neon` |
| Repository | `/run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad` |
| Filesystem boundary | `--one-file-system` |
| Required exclusions | `/home/neon/.cache` and `/home/neon/.local/share/Trash` |
| Downloads | Included |
| Baseline tag | `manual-baseline` |
| Password input | Interactive; no environment variable or plaintext password file |
| Integrity baseline | Structural check plus a 5% data sample |
| Restore proof | A selected path restored to a new private directory and compared |
| Retention | Not activated |
| Scheduling | Not activated |
| Recovery bundle | Raw LUKS2 header, complete sbctl state, EFI exports, configuration, inventory, and checksums |
| Installation media | A current verified Arch ISO on a physically available USB drive |

The paths describe the first ThinkPad as installed. They are not universal
examples to paste into another machine. The external mount must be identified
again on every run, and the future second ThinkPad should receive a distinct
repository identity rather than writing blindly into this path.

## Different safeguards answer different failures

Several mechanisms in this project preserve information, but they are not
substitutes for one another:

| Mechanism | Protects against | Does not protect against |
| --- | --- | --- |
| Restic snapshots | Deleted or changed home files, internal-disk loss when the repository survives | Loss of every repository copy or password; omitted source data; deletion by an attacker with repository access |
| Git remote | Reviewed committed history in the four public repositories | Uncommitted work, personal documents, browser state, credentials, or ignored files |
| File synchronization | Makes current files available in another location | Propagating deletion, corruption, or ransomware; it is not version retention by itself |
| LUKS2 | Offline disclosure after theft of the internal SSD | Deletion, filesystem damage, a logged-in attacker, or SSD failure |
| RAID or a mirrored disk | Some device failures and availability interruptions | User error, malicious deletion, bad updates, or a second independent copy |
| Filesystem snapshot | Fast point-in-time state on the same storage stack | Loss of that storage stack unless replicated elsewhere |
| Recovery bundle | Boot trust, encryption metadata, and a system inventory | Ordinary user data |
| Arch ISO | A bootable environment for unlock, mount, chroot, and repair | The workstation's data, credentials, or configuration history |

The resulting design is layered. Git makes the documented system reproducible;
Restic preserves user state; the raw bundle preserves exceptional identity and
boot material; the ISO provides tools when the installed operating system does
not start.

## Threat model and the present 3-2-1 gap

The usual 3-2-1 objective means three copies of important data, on two kinds
of storage, with one copy physically separate. Counting must be done per data
set, not per device:

| Data set | Current copies | Remaining gap |
| --- | --- | --- |
| Published repositories | Working clone plus GitHub remote | Local uncommitted work still depends on the home backup |
| Recovery bundle | Encrypted external disk plus required second encrypted copy | Both must actually be maintained after key changes |
| Restic-protected home | Internal live home plus first external repository | Irreplaceable data still needs a second, physically separate repository or equivalent copy |
| Restic password | Human memory is not counted; separate tested record required | The record must remain reachable if the laptop and backup disk are both unavailable |
| Arch recovery environment | Installed system plus physical ISO USB | ISO freshness and verification must be revisited periodically |

The first external repository improves recovery but does not complete 3-2-1.
If the laptop and backup disk share one bag, desk, fire, theft event, or power
incident, they share a failure domain.

Encryption also changes the recovery obligation. It protects confidentiality,
but makes every usable password or key record critical. An encrypted backup
whose only password is inside that backup has no independent recovery path.

## The protection map

The project deliberately keeps four classes of material separate:

### User data in Restic

The `/home/neon` snapshot contains documents, project working trees,
application configuration, application data, histories, browser profiles, and
the user's keyring unless excluded by a later reviewed policy. The repository
encrypts this material independently of the external filesystem.

### Exceptional raw recovery material

`rogue-thinkpad-recovery` contains material that should not be placed casually
inside the everyday home snapshot:

- the LUKS2 header and keyslot area;
- `/var/lib/sbctl`, including private signing keys and state;
- exported PK, KEK, db, and dbx firmware variables;
- selected boot and system configuration;
- storage, package, unit, and signature inventories;
- checksums of the complete bundle.

This directory must live on encrypted media that preserves Unix ownership and
permissions. Restic can store its encrypted repository on many filesystems,
but a raw root-owned secret bundle on exFAT, FAT32, or an unencrypted everyday
USB device does not have the same protection.

### Public reproducible configuration in Git

The four repositories preserve reviewed documentation and portable dotfiles.
They must contain no Restic password, LUKS header, Secure Boot private key,
browser profile, Secret Service database, or other private recovery material.

### Credentials and tools outside the failed machine

The Restic password and a valid LUKS passphrase need independent, tested
storage. The Arch ISO must be physically bootable. Keeping their only copies
inside `/home/neon`, GNOME Keyring, or the Restic repository creates a circular
dependency during recovery.

## Restic's repository model

### Repository, snapshot, and source tree

A Restic repository is not a mirror of `/home/neon`. It is a content-addressed
store containing encrypted objects and metadata. A backup scans the selected
source, divides file content into chunks, stores new content, and finally
writes a snapshot that describes one source tree at one time.

The main concepts are:

| Object | Role |
| --- | --- |
| Snapshot | A reference to a backed-up tree plus time, hostname, paths, tags, and related metadata |
| Tree | Directory structure and entries for a snapshot |
| Data blob | A deduplicated chunk of file content |
| Tree blob | A deduplicated chunk of directory metadata |
| Pack | Repository file that groups encrypted blobs for storage efficiency |
| Index | Mapping from blob identifiers to their location in packs |
| Key file | An encrypted record that permits a password to unlock the repository master key |
| Lock | Coordination record used to prevent incompatible concurrent operations |
| Cache | Local regenerable acceleration data, not the authoritative repository |

The snapshot is committed only after the data needed by it has been stored. If
the destination fills before completion, earlier snapshots remain valid and
some unreferenced data may remain, but there is no complete new snapshot to
pretend was successful.

### Deduplication is repository-local

If two snapshots contain an unchanged file, Restic normally reuses already
stored chunks instead of storing the whole file again. This happens by
content, not merely by filename. Renaming a file therefore need not duplicate
all of its contents.

Deduplication does not mean that a repository can be reconstructed from one
snapshot listing. Each retained snapshot can reference packs also used by
other snapshots. Deleting repository files manually bypasses the reference
model and can damage many restore points.

Independent repositories have independent encryption and chunking state. A
future second repository can be initialized with copied chunker parameters to
improve deduplication when using `restic copy`, but it remains a separate
repository with its own keys and failure boundary.

### A snapshot is not absolute immutability

Restic authenticates repository contents and detects unauthorized alteration,
but a writable backend or compromised client can still delete repository
objects. Snapshots can also be removed deliberately with `forget`, and their
unreferenced data can later be reclaimed with `prune`.

Therefore, “encrypted and authenticated” does not mean “undeletable”. A second
offline repository, restricted backend, append-only design, or provider-side
retention can add deletion resistance. None has yet been selected for this
project.

## Encryption and password ownership

### What the password unlocks

Restic generates repository encryption material and stores password-protected
key records. The password is not used to encrypt every data block directly.
This permits multiple passwords to unlock the same repository and permits a
password change without re-encrypting every stored snapshot.

Useful implications are:

- `restic key list` can show the repository's key records;
- `restic key passwd` can add a new password-protected key and remove the old
  record as part of a controlled password change;
- losing every valid password or usable decrypted key makes the encrypted
  content unrecoverable;
- learning one valid password grants access to all retained snapshots in that
  repository.

Changing a password does not revoke a repository key that an attacker already
decrypted and copied. That threat requires a new repository and a controlled
copy or fresh backup, not merely deleting one key record.

### Current manual password policy

The baseline prompts interactively:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad snapshots
```

It deliberately avoids:

- `--insecure-no-password`;
- a literal password in the command line;
- `RESTIC_PASSWORD` in a persistent session environment;
- a plaintext password file beside the repository;
- embedding a credential in a systemd unit or Git-tracked script.

Restic supports `--password-command` and `RESTIC_PASSWORD_COMMAND`, which may
eventually call a reviewed Secret Service helper. That is an interface, not an
automatic security improvement. The helper must return only the intended
secret, the keyring must be unlocked at timer time, logs must not expose it,
and GNOME Keyring cannot be the sole recovery record because its data resides
inside the home being restored.

## External media and repository identity

### A directory name does not prove that a disk is mounted

If a removable disk is absent, its former mount-point directory can still
exist on the internal filesystem. Writing a repository there could fill
`/home` or `/` while giving the operator the impression that an external
backup exists.

Before every manual operation, identify the actual block device and mount:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,TRAN
findmnt --target /run/media/neon/ARCH-BACKUP
df -hT /run/media/neon/ARCH-BACKUP
```

The `findmnt` source must be the expected external device, not the internal
root or home filesystem. A future automated job must compare a stable
filesystem UUID or another reviewed device identity; checking only that the
directory exists is insufficient.

Also inspect free space and repository placement:

```bash
findmnt --target /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad
df -hT /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad
```

The repository must not live beneath `/home/neon` itself. Backing a repository
into its own source would cause growth, recursion, and an invalid protection
boundary.

### One repository per ThinkPad

The simple future design is one repository per machine:

| Machine | Repository identity |
| --- | --- |
| Current installed ThinkPad | Existing `restic-rogue-thinkpad` |
| Future `t14-r5` | A separately initialized repository after the hostname and external-disk layout are finalized |
| Future `t14-r7` | A separately initialized repository after the hostname and external-disk layout are finalized |

Separate repositories isolate corruption, credentials, capacity, retention,
and administrative mistakes. They also make it obvious which machine can be
recovered from which store.

A shared repository is technically possible and would deduplicate common
content, but then hostname and path grouping become operationally critical. A
hostname change creates a new snapshot group by default; a careless retention
command could preserve one host while expiring another. This project prefers
the simpler independent boundary unless measured storage needs later justify a
shared design.

## Defining the source boundary

The canonical backup is:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    backup /home/neon \
    --one-file-system \
    --exclude '/home/neon/.cache' \
    --exclude '/home/neon/.local/share/Trash' \
    --tag manual-baseline
```

The continued lines above are Bash syntax on Arch. They are not PowerShell
commands for the Windows repository-management terminal.

### What the options mean

| Choice | Consequence |
| --- | --- |
| `/home/neon` | Selects the complete user home as the explicit source |
| `--one-file-system` | Does not cross into other mounted filesystems found below that source |
| Exclude `.cache` | Omits deliberately regenerable acceleration data |
| Exclude Trash | Omits files already intentionally placed in the desktop Trash |
| Include Downloads | Preserves downloads unless a future reviewed policy changes this |
| `--tag manual-baseline` | Marks the foundational snapshot so retention can preserve it deliberately |

`--one-file-system` is not a general “external disks are impossible” switch.
It applies while walking each explicitly supplied source. Supplying a second
source on another filesystem still asks Restic to back it up.

### Files that need special interpretation

| Source state | Restic behavior or risk |
| --- | --- |
| Symbolic link | Stored as a symbolic link; the target is not followed as ordinary file content |
| Socket | Not backed up as a restorable live socket |
| Owner-unreadable file | Reported as a source read error; the resulting snapshot is incomplete |
| Regenerable cache | Backed up unless explicitly excluded, consuming scan and storage work |
| Separate mount below home | Skipped by `--one-file-system` |
| Changing file during scan | Can be read at a different instant from related files |
| Database-like application state | May be application-inconsistent if changed while the scan is running |

Restic's snapshot is a repository snapshot, not an atomic ext4 snapshot of the
whole live home. Ordinary documents are normally suitable for a live scan,
but a multi-file database may require its application's export or quiescing
procedure. Restic can capture command output with `--stdin-from-command`; that
interface checks the producer's exit status. A raw shell pipeline into
`--stdin` can hide failure of the upstream command and should not be adopted
casually.

Restic preserves the metadata it supports, but it is not a block-level image
of ext4. For example, some filesystem-specific inode flags or creation-time
details are outside its portable restore model. The recovery design therefore
reinstalls the operating system and restores user data rather than expecting a
byte-identical disk clone.

### Exclusions are part of the backup policy

An exclusion saves time only by making data unrecoverable from that snapshot.
Before adding one, answer all of these questions:

1. Can the entire path be regenerated without the failed machine?
2. Are credentials, downloads, local-only edits, or application databases
   hidden below it?
3. Will the same exclusion still be correct on the second ThinkPad?
4. Is the exclusion documented next to the backup command?
5. Has a restore drill proved that the remaining data is sufficient?

Do not exclude broad paths such as `.local`, `.config`, browser profiles, or
all of `Downloads` merely because they are large.

## Running the manual backup safely

### Preflight

Before connecting the repository to a write operation:

```bash
findmnt --target /run/media/neon/ARCH-BACKUP
df -hT /run/media/neon/ARCH-BACKUP
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad snapshots
```

Confirm:

- the mount source, filesystem, label, and capacity match the intended disk;
- the repository opens with the separately stored password;
- no previous backup, check, copy, forget, or prune operation is still active;
- important applications with database-like state are closed or exported;
- AC power and the remaining session time are adequate;
- the machine will not suspend or disconnect the disk during the operation.

Then run the published backup command and read its complete final output.

### Exit status is part of the result

Restic defines meaningful backup exit codes:

| Exit | Meaning | Project interpretation |
| --- | --- | --- |
| `0` | Backup completed successfully | Candidate successful run; still inspect the snapshot |
| `1` | Fatal error; no snapshot created | Failure |
| `3` | Source files could not all be read; incomplete snapshot created | Failure requiring investigation, not “mostly successful” |
| `10` | Repository does not exist | Likely wrong path or missing mount; stop |
| `11` | Repository is already locked | Identify the real operation before considering unlock |
| `12` | Password is incorrect | Check repository identity and credential; do not initialize over it |

An automated job must preserve these distinctions in its systemd result and
journal. Treating every created snapshot as success would hide exit `3` and
silently normalize missing files.

### Inspect the new snapshot

List snapshots grouped by their normal identity dimensions:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    snapshots --group-by host,paths
```

Record the exact snapshot ID printed for the new run. Inspect it instead of
assuming that `latest` always means the same host and source:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad ls SNAPSHOT_ID
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad stats SNAPSHOT_ID
```

`latest` is convenient in a single-host repository, but becomes ambiguous
when several hosts or path sets share a repository. For recovery and retention,
an explicit ID or explicit `--host` and `--path` filter is safer.

Useful inspection commands include:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad find important-file.txt
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad diff OLDER_ID NEWER_ID
```

`stats` describes repository data according to its selected mode; it is not a
substitute for checking the repository or restoring files.

## Integrity is a ladder, not one command

### Structural check

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad check
```

The default check validates repository structure, indexes, snapshots, trees,
and whether referenced blobs are known. It does not read and cryptographically
verify every byte in every data pack.

### Sampled data read

The post-install baseline used:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    check --read-data-subset=5%
```

This reads a random sample. Repeating `5%` does not guarantee eventual full
coverage because a later random sample can overlap an earlier one.

### Deterministic rotating subsets

A future maintenance calendar can divide the repository into deterministic
parts:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    check --read-data-subset=1/20
```

Later runs use `2/20` through `20/20`. Completing every numerator for the same
denominator covers the complete repository across the cycle. The schedule must
record which subset ran successfully; repeatedly running `1/20` verifies only
that part.

### Full data read

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    check --read-data
```

This reads all repository data and can be slow and I/O-intensive. It is
appropriate periodically, after suspicious storage behavior, or before
retiring another copy, not necessarily after every daily backup.

### Actual restore

A restore drill tests more than repository structure:

- the password is available;
- the repository can be read through its real storage path;
- the intended snapshot and path can be found;
- data can be written to a target filesystem;
- restored contents and metadata are usable;
- the operator understands the non-destructive procedure.

No check command replaces this step. Conversely, restoring one small path does
not prove that every pack in the repository is readable. The layers complement
one another.

## A safe restore drill

### Select an exact source

List the repository and choose one explicit snapshot ID:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad snapshots
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad ls SNAPSHOT_ID
```

Choose a known, non-secret path whose live contents can be compared. Do not
begin with the whole home directory.

### Restore into an empty private directory

```bash
mkdir -m 0700 /home/neon/restic-restore-test
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    restore SNAPSHOT_ID \
    --target /home/neon/restic-restore-test \
    --include /home/neon/Projects/CycloniteRDX/niri-dotfiles \
    --verify
```

`--verify` reads the restored files and verifies their content after writing.
If a different path is chosen, replace the include and comparison paths
deliberately.

For a larger proposed restore, preview selection without writing:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    restore SNAPSHOT_ID \
    --target /home/neon/restic-restore-test \
    --include /home/neon/Documents \
    --dry-run --verbose=2
```

Do not use `--delete` in a drill. Do not target the live home root. Restoring
over existing application data can mix old and new state even when no command
reports an error.

### Compare and inspect

For the dotfiles example:

```bash
diff --recursive --brief \
    /home/neon/Projects/CycloniteRDX/niri-dotfiles \
    /home/neon/restic-restore-test/home/neon/Projects/CycloniteRDX/niri-dotfiles
```

No output means that the file contents compared by `diff` match. Also inspect
ownership, mode, links, and a representative file:

```bash
find /home/neon/restic-restore-test -maxdepth 4 -printf '%M %u:%g %y %p\n' | head -100
```

After review, remove the test through the graphical file manager so ordinary
Trash recovery remains available. Never add the restored tree to Git or leave
it beneath the next home backup indefinitely.

## Retention and space reclamation

### `forget` and `prune` are different operations

| Operation | Changes | Main risk |
| --- | --- | --- |
| `forget` | Removes selected snapshot references | The selected restore points disappear |
| `prune` | Reclaims repository data no retained snapshot references | Repacking, index changes, deletion, I/O load, and additional scratch-space needs |

Forgetting a snapshot does not immediately guarantee equivalent disk-space
recovery because its chunks may be shared with retained snapshots. Pruning is
the later maintenance operation that makes unreferenced data reclaimable.

Do not begin with `forget --prune`. Separate backup, inspection, dry-run
retention, actual forgetting, checking, pruning, and checking again so that
each mutation has an observable boundary.

### Snapshot grouping controls policy scope

Restic normally groups retention by `host,paths`. That prevents a daily policy
for one machine and source from automatically standing in for a snapshot of a
different machine or source.

A hostname change, source path change, or shared repository creates another
group. Always inspect groups before applying policy:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    snapshots --group-by host,paths
```

Do not simplify `--group-by` merely to make old snapshots disappear from a dry
run. First determine why the identity changed.

### Candidate policy, not an activated policy

After several weeks of measured snapshot frequency and repository growth, the
following may be evaluated:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad \
    forget --dry-run \
    --group-by host,paths \
    --keep-within 14d \
    --keep-daily 7 \
    --keep-weekly 8 \
    --keep-monthly 12 \
    --keep-yearly 3 \
    --keep-tag manual-baseline
```

This command is deliberately a dry run. It does not authorize removing the
snapshots it prints.

Interpret the rules carefully:

- `--keep-within 14d` preserves all snapshots whose timestamps are within the
  recent window;
- `--keep-daily 7` keeps snapshots from seven days that have snapshots, not a
  promise that seven consecutive calendar days exist;
- weekly, monthly, and yearly rules preserve representatives from their
  respective periods;
- matching keep rules are combined rather than applied as a simple deletion
  cascade;
- `--keep-tag manual-baseline` protects the tagged foundational snapshot;
- policy is evaluated separately for each chosen group.

Before any real `forget`, export or record the dry-run output, verify the exact
repository and groups, confirm the second copy, and run a current restore
drill. Pruning is reviewed separately and never runs automatically in the
initial design.

## A second repository and off-site copy

The second physically separate copy should be another independently encrypted
Restic repository or an equivalently reviewed backup. No disk, NAS, SFTP host,
or object-storage provider has yet been selected.

Restic can copy snapshots between repositories. When creating a new
destination intended to receive copied snapshots, copying chunker parameters
from the source can preserve more deduplication:

```bash
restic --repo DESTINATION init \
    --from-repo SOURCE \
    --copy-chunker-params
```

After initialization, a reviewed copy operation can select snapshots:

```bash
restic --repo SOURCE copy --repo2 DESTINATION
```

These are conceptual placeholders, not commands to run with the literal words
`SOURCE` and `DESTINATION`. Both repositories need their own accessible
credentials. Restic reads and decrypts source data, then encrypts it for the
destination; separate repositories retain separate keys and failure domains.

The copy must be checked and restored independently. Merely completing
`restic copy` does not prove that the destination password is stored elsewhere
or that the off-site backend can be reached during a laptop-loss event.

An append-only remote can reduce deletion risk from a compromised backup
client, but then retention and prune need a distinct trusted administrative
path. A `--keep-within` safety window is especially important because old data
may become deletable only after that window. This design requires a selected
backend and is not part of the present local-disk baseline.

## Why scheduling remains deferred

Restic does not provide its own scheduler. A systemd user service and timer can
eventually own the recurring operation, but only after these gates are solved:

| Gate | Required design |
| --- | --- |
| Device identity | Verify the actual mounted external filesystem by stable identity, not directory existence |
| Credentials | Use a reviewed non-logging password command; retain an independent recovery record |
| Session state | Define what happens when GNOME Keyring is locked or no graphical session exists |
| Power | Prefer AC power for long reads and define behavior after a missed run |
| Suspend | Inhibit sleep and shutdown while repository writes or checks are active |
| Capacity | Check free space and surface low-space failure before pruning anything |
| Concurrency | Prevent overlapping manual and scheduled maintenance and respect repository locks |
| Result | Treat exit `3` and every nonzero code as visible failure in the user journal |
| Disk absence | Fail safely without creating a repository on the internal filesystem |
| Retention | Keep forget and prune out of the initial automated backup path |
| Observability | Record start, selected repository identity, snapshot ID, exit status, and duration without secrets |

Relevant systemd building blocks include:

- `ConditionACPower=` for an AC-power condition;
- a timer with a deliberately chosen calendar and missed-run policy;
- `Persistent=true` only after understanding that a missed job may run soon
  after the timer becomes active again;
- `systemd-inhibit --what=sleep:shutdown` around the real backup operation;
- the user journal for inspection with `journalctl --user`.

The eventual automatic-suspend policy makes inhibition mandatory. The machine
must not suspend after idle timeout while Restic is updating packs or indexes.
Conversely, a permanently stuck inhibitor must be visible and diagnosable.

No unit is included here because choosing names and a timer before solving the
mount and credential boundaries would automate a false sense of safety.

## Maintaining the raw recovery bundle

The Restic repository does not replace
`rogue-thinkpad-recovery`. Refresh the raw bundle after any of these events:

- a LUKS passphrase, keyslot, token, or header change;
- regeneration, replacement, or re-enrollment of Secure Boot keys;
- a material ESP, UKI, boot-loader, kernel-command-line, or mkinitcpio change;
- a storage-layout change involving partition, LUKS, LVM, filesystem, or swap
  identity;
- a significant package or enabled-system-service baseline change.

After refreshing it:

1. recreate checksums from the known directory;
2. restrict the complete tree to root;
3. verify the LUKS backup header as a file with `luksDump`;
4. compare its UUID with the live container;
5. verify every checksum;
6. update the second encrypted, physically separate copy.

Never run `luksHeaderRestore` as a test. It writes encryption metadata to the
target device. A read-only inspection of the backup file is the safe routine
test.

The header backup is unusually sensitive: a backup header plus a passphrase
that was valid for it may retain access even after later keyslot changes. Key
rotation therefore requires controlling old header copies, not just changing
the live header.

## Recovery media is a maintained dependency

A current Arch ISO should be:

- downloaded from the official Arch site;
- verified with the published PGP signature or checksum procedure;
- written to a dedicated USB drive;
- booted in UEFI mode on the ThinkPad;
- proven able to detect the NVMe and network hardware;
- proven able to unlock LUKS, activate `vg0`, mount root, home, and ESP, and
  enter `arch-chroot`;
- stored where it remains reachable when the laptop does not boot.

An official ISO may not be signed by this machine's custom Secure Boot trust.
If firmware rejects it, temporarily disable Secure Boot for the recovery boot.
Do not clear enrolled keys or enter Setup Mode merely to start the ISO.

The ISO rehearsal is intentionally read-only with respect to installed boot
and encryption state. Successful chroot access proves a recovery route; it is
not permission to reinstall systemd-boot, enroll keys, rebuild UKIs, restore a
LUKS header, or update packages during the rehearsal.

## Recovery sequences

### One deleted or overwritten user file

1. Stop the application that might continue writing the path.
2. Mount and verify the intended repository disk.
3. Find the file and select an explicit snapshot ID.
4. Restore into a new private staging directory with `--verify`.
5. Inspect and compare the staged version.
6. Copy only the reviewed file back to its live location.
7. Keep the staging copy until the application opens the recovered data.

Do not restore the entire home over the live session for one missing file.

### Internal SSD failure

The safe high-level order is:

1. Preserve the failed device; do not repartition or write repairs to it while
   diagnosis or professional recovery remains possible.
2. Obtain replacement storage and the independent credentials, recovery
   bundle, Restic repository, Git access, and verified Arch ISO.
3. Reinstall the base system from `arch-linux-runbook` on the replacement disk.
4. Apply `arch-linux-post-install` in order, adapting only reviewed hardware or
   identity values.
5. Clone the four public Git repositories and deploy the portable dotfiles.
6. Open the Restic repository and inspect its snapshots before writing user
   data.
7. Restore the required home data into a staging tree, not blindly over an
   active new home.
8. Reconcile configuration and application state deliberately, preserving the
   new system's ownership and active session boundaries.
9. Restore Secure Boot identity or other raw bundle material only if the
   recovery plan actually requires continuity of those keys.
10. Verify applications, credentials, UKIs, Secure Boot, backup operation, and
    another restore before retiring remaining recovery copies.

The raw disk does not need to be cloned to reproduce the workstation. The
runbook rebuilds the known architecture; Git restores public configuration;
Restic restores user state.

## Repository trouble and safe diagnosis

### Locks

A lock normally means another Restic operation is protecting the repository.
First inspect running commands and storage activity. Only when the originating
process is certainly gone should a stale lock be removed:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad unlock
```

Do not use `unlock --remove-all` as a generic fix. It can remove active locks
and permit incompatible operations to overlap.

### Check errors or suspicious storage

When `check` reports damage:

1. stop backup, forget, prune, copy, and other repository writes;
2. preserve the entire repository storage if possible;
3. record the exact Restic version, command, output, mount, device identity,
   and kernel storage errors;
4. investigate the external disk, cable, enclosure, filesystem, and journal;
5. run only the diagnostic command justified by the error;
6. consult Restic's current troubleshooting guidance before any repair;
7. prefer restoring or copying from a known-good second repository.

Do not run `repair index` merely because a check failed. Restic's repair tools
address specific damage classes and can discard references to missing data.
Preserve at least `index` and `snapshots` before a reviewed repair if a complete
repository copy is impossible.

### Failure table

| Symptom | First checks | Unsafe first response |
| --- | --- | --- |
| Repository not found | `findmnt`, source device, exact path, transport connection | Run `restic init` at the same-looking directory |
| Wrong password | Repository identity, keyboard layout, independent password record | Create a new repository over the path |
| Backup exit `3` | Full output, unreadable files, disappearing mounts, application state | Accept the snapshot as complete |
| Repository locked | Running Restic processes and active maintenance | `unlock --remove-all` |
| Destination full | Stop mutation, preserve old snapshots, inspect capacity and duplicated data | Immediately forget and prune the only copy |
| `check` reports errors | Preserve repository, hardware logs, exact error class, second copy | Blind repair or prune |
| Expected file absent | Exact source path, exclusions, snapshot host/path/tag, backup date | Overwrite the live home with another snapshot |
| Restore differs | Selected snapshot, application consistency, symlinks, permissions, live changes | Delete the original before understanding the difference |
| Timer did not run | User manager, timer state, mount, keyring, AC condition, journal | Assume the external disk contains a current backup |

## Decisions recorded for this project

- Restic protects the complete user home except the two explicit regenerable
  paths; broad convenience exclusions are rejected.
- The raw recovery bundle remains separate from the routine Restic repository.
- The repository password remains interactive until an independently
  recoverable Secret Service design is reviewed.
- A created incomplete snapshot with exit `3` is a failed backup.
- Exact snapshot IDs are preferred for drills and recovery.
- Structural checks, rotating data reads, and real restores are separate proof
  layers.
- The proposed retention expression is a dry-run candidate, not active policy.
- Forget and prune will not be combined in the first maintenance workflow.
- Each ThinkPad should use a separate repository unless a later measured need
  justifies shared-host complexity.
- A second physically separate user-data copy remains required; no backend has
  yet been selected.
- Automatic backup waits for stable mount identity, safe credential retrieval,
  visible failure, power handling, and suspend inhibition.
- LUKS header restore, repository repair, repartitioning, and boot-key
  replacement are recovery-event actions, never routine tests.

## Verification checklist

- [ ] The external mount resolves to the intended device, not `/` or `/home`.
- [ ] The repository opens with a password stored independently of the laptop
      and repository disk.
- [ ] The current backup command still covers all of `/home/neon` except the
      two documented exclusions.
- [ ] The most recent backup returned `0`; any exit `3` was investigated.
- [ ] Its exact snapshot ID, hostname, path set, time, and tags were inspected.
- [ ] A structural `check` succeeds.
- [ ] Data-read coverage follows a recorded sample, rotating-subset, or full
      plan rather than repeated untracked random samples.
- [ ] A selected path restores into an empty `0700` directory with `--verify`.
- [ ] Restored data has been inspected and compared before live replacement.
- [ ] No retention command has been activated from the candidate dry run.
- [ ] The recovery bundle checksums verify and its header UUID matches the live
      LUKS container.
- [ ] A second encrypted copy of the recovery bundle exists elsewhere.
- [ ] Irreplaceable user data has or is scheduled to receive a second
      physically separate copy.
- [ ] A current verified Arch ISO can unlock, mount, and chroot into the
      installed architecture.
- [ ] No password, private key, LUKS header, or secret bundle has entered Git.

## Sources

- [Restic documentation](https://restic.readthedocs.io/en/latest/)
- [Preparing a new Restic repository](https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html)
- [Backing up](https://restic.readthedocs.io/en/latest/040_backup.html)
- [Working with repositories and `restic copy`](https://restic.readthedocs.io/en/latest/045_working_with_repos.html)
- [Restoring from backup](https://restic.readthedocs.io/en/latest/050_restore.html)
- [Removing snapshots and reclaiming space](https://restic.readthedocs.io/en/latest/060_forget.html)
- [Restic encryption](https://restic.readthedocs.io/en/latest/070_encryption.html)
- [Scripting Restic](https://restic.readthedocs.io/en/latest/075_scripting.html)
- [Restic troubleshooting and repair](https://restic.readthedocs.io/en/latest/077_troubleshooting.html)
- [`restic-backup(1)`](https://man.archlinux.org/man/restic-backup.1.en)
- [`restic-check(1)`](https://man.archlinux.org/man/restic-check.1.en)
- [`restic-restore(1)`](https://man.archlinux.org/man/restic-restore.1.en)
- [`restic-forget(1)`](https://man.archlinux.org/man/restic-forget.1.en)
- [`restic-unlock(1)`](https://man.archlinux.org/man/restic-unlock.1.en)
- [Restic package for Arch Linux](https://archlinux.org/packages/extra/x86_64/restic/)
- [ArchWiki system backup overview](https://wiki.archlinux.org/title/System_backup)
- [`cryptsetup-luksHeaderBackup(8)`](https://man.archlinux.org/man/cryptsetup-luksHeaderBackup.8.en)
- [Official Arch Linux installation media](https://archlinux.org/download/)
- [Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [`systemd.timer(5)`](https://man.archlinux.org/man/systemd.timer.5.en)
- [`systemd.unit(5)`](https://man.archlinux.org/man/systemd.unit.5.en)
- [`systemd-inhibit(1)`](https://man.archlinux.org/man/systemd-inhibit.1.en)

## Next guide

Guide 20 will close the essential handbook edition by connecting journal-led
diagnosis, update incidents, boot and chroot recovery, and the recurring
maintenance workflow. It will show how to choose the least destructive
recovery layer before changing a damaged system.
