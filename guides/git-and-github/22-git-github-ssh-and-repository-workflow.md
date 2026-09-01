# Git, GitHub, SSH authentication, and repository workflow

## Purpose and scope

Git protects reviewed history only when its model is understood. GitHub stores
and collaborates on published Git objects, but it does not automatically save
an edited working tree, an untracked file, an ignored secret, or a commit that
has never been pushed.

This guide explains:

- the difference between Git, GitHub, SSH, and GitHub CLI;
- the working tree, index, commits, branches, remotes, and remote-tracking
  branches;
- author identity, SSH host verification, user authentication, and repository
  authorization as separate concerns;
- the project's Windows/PowerShell and Arch/Bash workflows;
- repository creation, cloning, remote URLs, and first publication;
- deliberate staging, review, Conventional Commits, pull, and push;
- feature branches without adding ceremony to small documentation changes;
- conflicts, diverged history, stashes, reflogs, reverts, and safe recovery;
- line endings, executable bits, secrets, and coordination across four
  repositories.

It does not make repositories public or private, create or delete GitHub
repositories, upload an SSH private key, expose personal key fingerprints,
enable an SSH server, rewrite published `main` history, install a graphical Git
client, or define an organization-wide collaboration policy.

## Current project contract

The examples reflect the current project rather than an abstract repository:

| Concern | Current decision |
| --- | --- |
| GitHub account and Git author name | `CycloniteRDX` |
| Commit email | `219207771+CycloniteRDX@users.noreply.github.com` |
| Primary authoring environment | VS Code on Windows 11, using PowerShell |
| Installed workstation | Arch Linux, using Bash and OpenSSH |
| Canonical branch | `main` |
| Conventional remote name | `origin` |
| Authoring transport | SSH |
| Fresh public clones on Arch | HTTPS is acceptable for read-only retrieval |
| Ordinary publication | Git over SSH; no personal access token is needed |
| GitHub CLI | Platform/API operations only, not a replacement for Git |
| Server exposure | `sshd.service` stays disabled; outgoing SSH needs no SSH server |
| Repository layout | Four independent repositories, not one umbrella repository |
| Review policy | Inspect status and diffs, stage exact paths, then commit and push |

The four repositories have distinct responsibilities:

| Repository | Responsibility |
| --- | --- |
| `arch-linux-runbook` | Installation from firmware and ISO to the first TTY |
| `arch-linux-post-install` | Operational construction of the workstation |
| `arch-linux-handbook` | Concepts, trade-offs, diagnosis, and recovery |
| `niri-dotfiles` | Portable user configuration deployed with GNU Stow |

A change spanning two repositories therefore requires two reviewed commits.
There is no atomic cross-repository commit. Publish the implementation first
and the documentation that depends on it second when ordering matters.

## Git, GitHub, SSH, and `gh` are different layers

| Layer | What it does | What it does not do |
| --- | --- | --- |
| Git | Creates and relates local snapshots; compares and transfers objects | Host an account, issue tracker, or pull-request website |
| GitHub | Hosts Git repositories and collaboration metadata | Save uncommitted local edits automatically |
| SSH | Authenticates a client and encrypts a transport connection | Define Git authorship or commit history |
| GitHub CLI (`gh`) | Calls GitHub's web API and assists with platform operations | Replace `git add`, `git commit`, `git fetch`, or `git push` |

An ordinary SSH push follows several independent decisions:

1. Git reads the repository and determines which objects need transfer.
2. The local SSH client verifies the server's host identity.
3. GitHub verifies that the client controls an account-associated private key.
4. GitHub authorizes that account for the requested repository and branch.
5. Git transfers objects and asks the remote to update a reference.

A failure at one layer should not be treated as proof that every layer is
wrong. A correct commit author email does not authenticate SSH; a valid SSH
key does not grant access to every repository; successful `gh auth status`
does not prove the Git remote uses SSH.

## The repository model

### Working tree, index, commit, and remote

A normal non-bare repository contains:

| Object or view | Meaning |
| --- | --- |
| `.git/` | Git's object database, references, local configuration, and metadata |
| Working tree | The checked-out files being inspected and edited |
| Index, or staging area | The proposed content of the next commit |
| `HEAD` | The currently checked-out commit or branch reference |
| Local branch | A movable name such as `main` pointing at one commit |
| Remote-tracking branch | The last fetched view, such as `origin/main` |
| Remote | A named transfer destination such as `origin` |

The index is not merely a list of filenames. It stores the exact content and
metadata proposed for the next commit. A file may therefore contain both
staged and unstaged changes at the same time.

The most useful comparison map is:

| Command | Comparison |
| --- | --- |
| `git diff` | Working tree versus index |
| `git diff --cached` | Index versus `HEAD` |
| `git diff HEAD` | Working tree plus index versus `HEAD` |
| `git show HEAD` | The current commit |
| `git diff main...topic` | Changes introduced on `topic` since the merge base |

### Commits are snapshots connected by parents

A commit records:

- a complete project tree;
- one or more parent commit identities;
- author and committer metadata;
- a message.

Git internally reuses unchanged objects, but the useful operator model is a
snapshot, not a bag of independent file patches. The commit identifier covers
the recorded content and metadata. Amending a message, author, tree, or parent
creates a different commit identifier.

A branch is a movable reference to a commit. It is not a second directory or
a server-side copy. `HEAD` normally points to the current branch, and the
branch advances when a new commit is made.

### What Git does not protect

Git cannot normally restore:

- an untracked file deleted before it was added;
- an unstaged edit discarded with `git restore`;
- ignored material that was never committed;
- a local commit after its reflog and objects have expired;
- a local commit that was never pushed if the local repository is lost.

Git is version control, not a complete backup. Git remotes protect only pushed
objects that the server retains. Guide 19 owns data backup and restore drills.

## Identity, authentication, and authorization

Four identities are commonly confused:

| Identity | Evidence | Question answered |
| --- | --- | --- |
| Git author/committer | `user.name` and `user.email` stored in commits | Who does this commit claim made it? |
| SSH server | Host key checked against `known_hosts` | Is this really the intended GitHub server? |
| SSH client/account | Proof using the private key whose public key GitHub stores | Which GitHub account is connecting? |
| GitHub authorization | Repository and branch permissions | May this account perform this push? |

A commit signature is a fifth, optional mechanism. It cryptographically signs
a commit or tag; it is not the same operation as using SSH to transport an
unsigned commit. This project has not selected a commit-signing policy.

## Inspect and set Git configuration deliberately

Git configuration can come from system, global, local-repository, worktree,
and command scopes. Inspect both the value and its origin:

```powershell
git config --list --show-origin --show-scope
git config --get-all user.name
git config --get-all user.email
git config --get-all init.defaultBranch
git config --get-all pull.ff
```

For this project on the normal authoring account:

```powershell
git config --global user.name "CycloniteRDX"
git config --global user.email "219207771+CycloniteRDX@users.noreply.github.com"
git config --global init.defaultBranch main
git config --global pull.ff only
```

Verify the effective values from inside a repository:

```powershell
git config --show-origin --get user.name
git config --show-origin --get user.email
git config --show-origin --get init.defaultBranch
git config --show-origin --get pull.ff
```

The noreply address reduces exposure of the account's personal email in new
commit metadata. It does not rewrite earlier commits, prove account ownership,
or make a public commit private.

### Settings deliberately not imposed

- Do not set a wildcard `safe.directory`. Resolve the actual ownership problem
  or trust one exact inspected repository when genuinely necessary.
- Do not force a global `core.sshCommand` under normal conditions. Git should
  use the OpenSSH client found in the current environment unless a diagnosed
  client mismatch requires an explicit local decision.
- Do not use `core.autocrlf` to override repository policy. The project's
  `.gitattributes` declares LF for text and marks known binary types.
- Do not store credentials, tokens, or private-key paths inside repository
  configuration.
- Do not set an automatic merge strategy merely to hide divergence. `pull.ff
  only` makes unexpected divergence stop for review.

## SSH authentication to GitHub

### Private key, public key, and host key

These are different:

- the private key remains on the client and must never be committed, uploaded,
  pasted into an issue, or sent to GitHub;
- the public key may be registered on GitHub;
- GitHub's host key identifies the server to the client;
- a passphrase encrypts the private-key file at rest;
- an SSH agent can hold an unlocked key for a session.

The established Windows authoring machine already has a verified Ed25519 key
that authenticates to GitHub. Do not generate another key merely because a
generic tutorial begins with `ssh-keygen`, and do not publish its personal
fingerprint in this handbook.

### Inspect an existing key before creating anything

PowerShell:

```powershell
Get-ChildItem -Force "$HOME\.ssh"
ssh-keygen -lf "$HOME\.ssh\id_ed25519.pub"
ssh-add -l
```

Bash on Arch:

```bash
ls -la -- "$HOME/.ssh"
ssh-keygen -lf "$HOME/.ssh/id_ed25519.pub"
ssh-add -l
```

`ssh-add -l` may report that it cannot connect to an agent or that the agent
has no identities. That does not prove the key files are absent and does not
justify overwriting them.

Only on a genuinely new client, after confirming the target does not exist,
create a client-specific key:

```powershell
ssh-keygen -t ed25519 -C "219207771+CycloniteRDX@users.noreply.github.com"
```

Use a passphrase and accept the normal key path only if it is free. A distinct
key per important client improves revocation and attribution: losing one
laptop need not invalidate the keys on every other machine.

### Verify GitHub's host before accepting it

The first SSH connection may ask whether to trust GitHub's host key. Compare
the shown fingerprint with GitHub's currently published fingerprint over an
independently trusted HTTPS connection. As checked for this guide, GitHub
publishes this Ed25519 fingerprint:

```text
SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU
```

Do not treat `ssh-keyscan` output obtained through the same untrusted network
as independent proof of the host. Recheck the official
[GitHub SSH-key fingerprint page](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)
when configuring a new machine; host keys can change.

Test authentication:

```powershell
ssh -T git@github.com
```

GitHub should identify the account and state that it does not provide shell
access. GitHub documents that this successful test can exit with status `1`
because no interactive shell is offered. Interpret the message, not that
status in isolation.

### Register only the public key

The public-key text can be copied from Windows:

```powershell
Get-Content "$HOME\.ssh\id_ed25519.pub"
```

Or from an active Arch Wayland session:

```bash
wl-copy < "$HOME/.ssh/id_ed25519.pub"
```

Add that public key in GitHub's SSH-key settings. Never copy the file without
the `.pub` suffix.

### Diagnose the actual SSH client

Windows can have more than one Git and OpenSSH installation. Inspect resolution
before adding client-specific configuration:

```powershell
Get-Command git
Get-Command ssh
git --version
ssh -V
git config --show-origin --get core.sshCommand
ssh -G git@github.com
```

For a bounded verbose test:

```powershell
ssh -vT git@github.com
```

Verbose output may expose usernames, filesystem paths, host aliases, agent
state, and public-key fingerprints. Redact it before sharing.

If a network blocks outbound TCP port 22, GitHub supports SSH through
`ssh.github.com` on port 443. That is a diagnosed fallback, not the default;
follow GitHub's current
[SSH-over-HTTPS-port instructions](https://docs.github.com/en/authentication/troubleshooting-ssh/using-ssh-over-the-https-port)
and verify the host identity before saving a host alias.

### Outgoing SSH does not require `sshd`

`ssh` is a client; `sshd` is a server. Cloning and pushing to GitHub initiate
outgoing connections and do not require:

- enabling `sshd.service`;
- opening inbound TCP port 22;
- adding an SSH service to firewalld.

The project's disabled SSH server and closed inbound firewall policy therefore
remain unchanged.

## Remote URLs and transport

Inspect the remote, including any distinct fetch and push URLs:

```powershell
git remote -v
git remote get-url --all origin
git remote get-url --push --all origin
```

Public HTTPS clones are appropriate when a fresh Arch installation only needs
to read the repositories:

```bash
git clone https://github.com/CycloniteRDX/arch-linux-handbook.git
```

An authoring clone can use SSH:

```powershell
git clone git@github.com:CycloniteRDX/arch-linux-handbook.git
```

To change one inspected repository from HTTPS to SSH:

```powershell
git remote set-url origin git@github.com:CycloniteRDX/arch-linux-handbook.git
git remote -v
ssh -T git@github.com
git fetch origin --prune
git status --short --branch
```

Replace the repository name with the exact repository being configured. Do
not paste a generic `REPOSITORY` placeholder literally.

`origin` is only a conventional local name. It is not the server, account, or
canonical branch. `origin/main` is a local remote-tracking reference updated
by fetch; it is not a live view that changes by itself.

## GitHub CLI without conflating it with Git

The installed `gh` client is useful for repository creation, pull requests,
issues, releases, and other GitHub API operations. It is not needed for
ordinary Git-over-SSH clone, fetch, pull, or push.

When GitHub CLI authentication is actually required on a machine that already
has its SSH key registered:

```powershell
gh auth login --hostname github.com --git-protocol ssh --web --skip-ssh-key
gh auth status --hostname github.com
```

`--skip-ssh-key` prevents the login assistant from trying to discover or
upload another key. The web flow avoids pasting a token into shell history.
GitHub CLI normally stores credentials in a secure credential store, but its
manual documents a plaintext configuration-file fallback if secure storage
fails. Stop and investigate rather than accepting that fallback silently.

Useful read-only checks include:

```powershell
gh auth status --hostname github.com
gh repo view CycloniteRDX/arch-linux-handbook
```

`gh auth logout` removes local authentication state; it does not necessarily
revoke every server-side token. Do not print `gh auth token` into a transcript,
script, issue, or repository.

## Clone an existing repository safely

Before cloning, inspect the intended parent directory and ensure no conflicting
directory exists.

PowerShell:

```powershell
Set-Location "D:\Marcos\Documentos\Embedded"
Get-ChildItem
git clone git@github.com:CycloniteRDX/arch-linux-handbook.git
Set-Location ".\arch-linux-handbook"
git remote -v
git status --short --branch
```

Bash on Arch:

```bash
mkdir -p -- "$HOME/Projects/CycloniteRDX"
cd -- "$HOME/Projects/CycloniteRDX"
git clone https://github.com/CycloniteRDX/arch-linux-handbook.git
cd -- arch-linux-handbook
git remote -v
git status --short --branch
```

A clone normally creates `origin`, fetches objects, checks out the remote's
default branch, and configures its upstream relationship. Verify; do not infer
success merely from the directory existing.

Do not:

- extract a ZIP containing another repository's `.git` directory;
- initialize Git in a parent that unintentionally contains all four projects;
- initialize a nested repository inside an existing one;
- copy `.git` as a project template;
- use `--allow-unrelated-histories` to join two independently initialized
  histories without a deliberate migration plan.

Locate the current repository root:

```powershell
git rev-parse --show-toplevel
git status --short --branch
```

If `git rev-parse` reports that the directory is not a repository, stop and
locate the intended clone. Do not run `git init` merely to silence the error.

## Create a new repository without unrelated histories

Prefer one of two clean flows:

1. create an empty repository on GitHub, initialize locally, add that remote,
   and push; or
2. create the GitHub repository with its initial files, clone it, and continue
   in that clone.

Do not independently create a README commit on both sides and then try to
force them together.

For an empty remote, the local PowerShell sequence is:

```powershell
New-Item -ItemType Directory -Path ".\new-project"
Set-Location ".\new-project"
git init --initial-branch=main
git status --short --branch
```

Create and review `README.md`, `.gitignore`, and `.gitattributes` using the
editor. Then:

```powershell
git add -- README.md .gitignore .gitattributes
git diff --cached --check
git diff --cached
git commit -m "chore(repo): initialize project"
git remote add origin git@github.com:CycloniteRDX/new-project.git
git remote -v
git push -u origin main
git status --short --branch
```

The GitHub repository must already exist, be empty, and have the intended
visibility. Repository creation and visibility are explicit GitHub-side
actions; this handbook never assumes authorization to perform them.

## The canonical daily workflow

Run every command from the exact repository being changed. PowerShell commands
are kept on one line because Bash's trailing backslash is not PowerShell line
continuation.

### 1. Establish location and starting state

```powershell
git rev-parse --show-toplevel
git status --short --branch
git remote -v
```

On a clean, synchronized canonical branch:

```text
## main...origin/main
```

No file lines should follow it.

### 2. Fast-forward before editing

When the tree is clean:

```powershell
git pull --ff-only
git status --short --branch
```

`--ff-only` updates the branch only when no merge commit or rebase decision is
needed. A refusal is valuable evidence of divergence, not an inconvenience to
override.

If local work already exists, inspect it first. Do not hide it with an
automatic stash or mix an upstream update into an unknown working tree.

### 3. Inspect all edits

```powershell
git status --short
git diff --check
git diff --stat
git diff
```

`git diff --check` detects whitespace errors, not factual, syntactic, or
semantic correctness. Use format-specific validators as well.

### 4. Stage exact paths

```powershell
git add -- guides/README.md guides/git-and-github/22-git-github-ssh-and-repository-workflow.md
```

The `--` ends option parsing, so a path beginning with a hyphen is not treated
as an option. Exact paths make the commit boundary visible. Avoid `git add .`
when unrelated local files may exist.

For selected hunks in one file:

```powershell
git add -p -- path/to/file
```

Read every hunk. Partial staging can be useful, but it can also stage a state
that has never existed in the working tree or passed its validator.

### 5. Review the proposed commit

```powershell
git diff --cached --check
git diff --cached --stat
git diff --cached
git status --short
```

The first status column describes the index; the second describes the working
tree:

| Example | Meaning |
| --- | --- |
| `M  file` | Modified content is staged |
| ` M file` | Modified content is unstaged |
| `MM file` | One change is staged and another remains unstaged |
| `A  file` | A new file is staged |
| `D  file` | Deletion is staged |
| `?? file` | Untracked; absent from the next commit |

The goal is not always an empty second column. The goal is to know exactly
which state the proposed commit contains.

### 6. Validate and commit one logical change

```powershell
git commit -m "docs(handbook): explain Git and GitHub workflow"
git show --stat --oneline HEAD
```

A successful commit is local. It is durable in this repository but is not yet
published.

### 7. Push and prove synchronization

```powershell
git push
git status --short --branch
git log --oneline --decorate -5
```

The final expected status is again:

```text
## main...origin/main
```

`git push` does not publish uncommitted work, ignored files, arbitrary local
branches, or tags unless the refspec/configuration asks for them.

## Conventional Commits in this project

The project uses the following readable shape:

```text
type(scope): imperative summary
```

The full Conventional Commits shape also allows:

```text
type(scope)!: summary
```

with a `BREAKING CHANGE:` footer when an incompatible change truly exists.

Common project types:

| Type | Use |
| --- | --- |
| `docs` | Documentation-only change |
| `feat` | New user-visible configuration or capability |
| `fix` | Correction to behavior or configuration |
| `refactor` | Internal reorganization without intended behavior change |
| `test` | Verification material |
| `chore` | Repository maintenance that fits no stronger type |
| `build` | Build or packaging mechanics |
| `ci` | Continuous-integration policy |

Scopes name the affected responsibility, not the computer used:

```text
docs(runbook): explain fallback boot verification
docs(post-install): configure core workstation services
docs(handbook): explain Git and GitHub workflow
feat(niri): add minimal graphical bootstrap
feat(mimeapps): define default applications
feat(autostart): add removable-media automount
```

Prefer one logical purpose per commit. A message describes why this snapshot
belongs in history; it does not need to enumerate every changed filename.

### Author and committer metadata

Git records author and committer identities independently. They are usually
the same for direct work, but operations such as applying another person's
patch can differ. Changing `user.email` affects future commits; it does not
rewrite published history.

Inspect the current commit:

```powershell
git show --no-patch --format=fuller HEAD
```

Amend only an unpublished commit after inspecting the result:

```powershell
git commit --amend
git show --stat --format=fuller HEAD
```

After publication, prefer a new correction or `git revert`. An amend replaces
the commit identity and would require rewriting the remote branch.

## When to work directly on `main` and when to branch

The current single-operator chapter workflow permits one small, independently
reviewed documentation change directly on `main`. Use a topic branch when:

- the change needs several commits;
- the experiment may be abandoned;
- review should happen through a pull request;
- another person may update `main` concurrently;
- the change modifies risky configuration or automation.

Create and publish a branch:

```powershell
git switch -c docs/guide-22
git status --short --branch
git push -u origin docs/guide-22
```

GitHub CLI can then create a pull request only after its separate
authentication is configured:

```powershell
gh pr create --base main --head docs/guide-22
```

Do not run a force push to `main`. A force-with-lease may be considered only on
an unpublished or personal topic branch after inspecting who else depends on
it.

### Detached `HEAD`

Checking out a tag or raw commit can detach `HEAD`. New commits then do not
advance a normal branch and are easier to lose.

Inspect:

```powershell
git status --short --branch
git branch --show-current
```

If valuable work was committed while detached, preserve it immediately:

```powershell
git switch -c rescue/detached-work
```

## Fetch, pull, and push

| Operation | Main effect |
| --- | --- |
| `git fetch origin --prune` | Downloads remote objects and updates/removes remote-tracking refs; does not integrate into the current branch |
| `git pull --ff-only` | Fetches and advances the current branch only if a fast-forward is possible |
| `git push` | Sends required objects and requests remote-reference updates |

A fast-forward moves a branch reference along its existing ancestry. It
creates no merge commit and discards no side history.

Fetch is normally the safest first network diagnosis:

```powershell
git fetch origin --prune
git status --short --branch
git log --graph --decorate --oneline --all -20
```

Network success does not prove working-tree cleanliness, and a clean tree does
not prove synchronization. Check both.

## A rejected push and diverged history

A non-fast-forward push rejection protects remote commits that the proposed
update would remove from the remote branch. Do not answer it with a blind
force.

Inspect:

```powershell
git fetch origin --prune
git status --short --branch
git log --graph --decorate --oneline --all -30
git log --left-right --cherry-pick --oneline main...origin/main
```

Before rewriting unpublished local commits, preserve a readable pointer:

```powershell
git branch rescue/main-before-reconcile
```

For this project's single-operator, linear documentation history, rebasing
unpublished local commits onto the fetched `origin/main` is often appropriate:

```powershell
git rebase origin/main
```

If conflicts appear, resolve them deliberately as described below. To abandon
the operation:

```powershell
git rebase --abort
```

A merge is an alternative when preserving both development lines is
intentional:

```powershell
git merge origin/main
```

Do not choose between rebase and merge merely to make the warning disappear.
Rebase creates new identities for the replayed local commits; merge preserves
both lines and adds a merge commit.

## Resolve conflicts without guessing

A conflict means Git could not safely select one combined content state. It
does not necessarily mean one contributor was wrong.

During a merge or rebase:

```powershell
git status
git diff
```

Conflict markers contain these seven-character prefixes; the explanatory
labels below are not part of the file:

```text
start:     <<<<<<< current side
one side
separator: =======
other side
end:       >>>>>>> incoming side
```

Edit the file into the single intended result, remove every marker, validate
the whole file, and stage that exact path:

```powershell
git add -- path/to/resolved-file
git diff --cached --check
git status
```

Continue the active operation:

```powershell
git rebase --continue
```

or:

```powershell
git merge --continue
```

Abort if the intended result is unclear:

```powershell
git rebase --abort
```

or:

```powershell
git merge --abort
```

The meaning of “ours” and “theirs” can be counterintuitive during rebase
because commits are being replayed onto another base. Do not use blanket
checkout options without reading the actual content.

## Safe recovery map

Identify whether the content exists in the working tree, index, a commit, a
reflog, a stash, another clone, or nowhere before changing anything.

| Situation | Conservative response |
| --- | --- |
| Wrong file staged; working edit should remain | `git restore --staged -- path` |
| Tracked file deleted or changed only in working tree; discard is intended | Inspect, then `git restore -- path` |
| Untracked file deleted before commit | Git normally cannot recover it; check editor history, Trash, and backups |
| Useful local commit no longer named by a branch | Find it with `git reflog` and create a rescue branch |
| Last commit is wrong and unpublished | Amend carefully or make another local commit |
| Published commit should be undone | `git revert COMMIT`, review the new commit, then push |
| Work must move aside temporarily | Named stash including the intended file classes, or preferably a WIP branch |
| Merge/rebase is misunderstood | Abort the active operation and return to the pre-operation state |
| Secret was pushed | Revoke/rotate it first; deletion from the latest tree is insufficient |

### Unstage without discarding the working edit

```powershell
git restore --staged -- path/to/file
git status --short
git diff -- path/to/file
```

This changes the index. The working-tree edit remains.

### Discard an unstaged tracked edit

```powershell
git diff -- path/to/file
git restore -- path/to/file
```

The second command discards working-tree content and Git may have no recovery
path for it. Use it only after the diff proves that deletion is intended.

### Revert a published commit

```powershell
git show --stat COMMIT
git revert COMMIT
git show --stat --oneline HEAD
git push
```

`git revert` creates a new commit that reverses selected changes while
preserving the published history. It can conflict if later work depends on the
reverted content.

### Recover a lost local commit with the reflog

```powershell
git reflog --date=local
git show LOST_COMMIT
git branch rescue/recovered-work LOST_COMMIT
```

The reflog records local reference movements. It is local, can expire, and is
not a substitute for a remote or backup. Create the rescue branch before
attempting another reconciliation.

### Stash only with an explicit content boundary

Default `git stash` omits untracked files. `-u` includes untracked files; `-a`
also includes ignored files and can capture secrets or bulky generated data.

```powershell
git status --short
git stash push -u -m "wip: before upstream reconciliation"
git stash list
git stash show -p "stash@{0}"
git stash apply "stash@{0}"
git status --short
```

After verifying the applied result:

```powershell
git stash drop "stash@{0}"
```

For important or long-lived work, a named branch and normal WIP commit are
usually more inspectable than a stash.

### Commands deliberately excluded from routine recovery

Do not use these as generic cleanup:

- `git reset --hard`;
- `git clean -fd` or `git clean -fdx`;
- legacy `git checkout -- path` used to discard content;
- `git push --force`;
- deleting `.git` and recloning before preserving unique local work.

Each can be valid in a narrowly proven situation, but each can also make local
content difficult or impossible to recover.

## Ignore rules and secrets

`.gitignore` affects untracked-path discovery. It does not remove a file that
is already tracked.

Inspect why a path is ignored:

```powershell
git check-ignore -v -- path/to/file
```

Inspect whether it is already tracked:

```powershell
git ls-files --error-unmatch -- path/to/file
```

The repositories ignore editor and operating-system debris. `niri-dotfiles`
also reserves patterns and directories for local or secret material. An ignore
rule is convenience, not access control: a force-add, rename, archive, log, or
careless copy can still disclose a secret.

If a secret is only staged, unstage and relocate it before committing:

```powershell
git restore --staged -- path/to/secret
```

If it was committed and pushed:

1. revoke or rotate the credential immediately;
2. determine every exposed remote, fork, artifact, and log;
3. coordinate any required history rewrite;
4. replace affected clones and credentials;
5. document the incident privately.

Deleting it in a later commit does not erase it from earlier reachable
history.

## Cross-platform repository policy

### Line endings

Each repository's `.gitattributes` contains:

```gitattributes
* text=auto eol=lf

*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.webp binary
*.pdf binary
*.zip binary
```

This is shared repository policy: text is normalized to LF and known binary
files are not text-normalized. Inspect attributes and actual states:

```powershell
git check-attr -a -- path/to/file
git ls-files --eol
git diff --check
```

Do not “fix” a whole repository's line endings during an unrelated chapter.
If normalization is intentionally changed, make it an isolated reviewed
commit.

### Executable bits

Windows does not represent Unix executable permission in the same way as an
Arch worktree. For a reviewed script that must be executable after clone:

```powershell
git update-index --chmod=+x -- path/to/script
git diff --cached --summary
```

Do not mark documents or configuration data executable.

### Symbolic links and Stow

`niri-dotfiles` stores ordinary source files in package-shaped directories.
GNU Stow creates the user-home links on Arch. Windows is used to author and
publish the repository; it does not need to reproduce the deployed symlink
tree.

### Shell syntax

Bash and PowerShell are different languages:

- trailing `\` continues a command in Bash, not PowerShell;
- `/dev/null` is a Unix device path, not a PowerShell path;
- PowerShell commands in delivery packages stay on one line where practical;
- copied prompts such as `PS C:\>` or `$` are not part of the command.

Guide 21 owns the broader shell, path, quoting, and documentation model.

## Coordinating the four repositories

Before a cross-repository delivery, inspect every involved repository
independently:

```powershell
git rev-parse --show-toplevel
git status --short --branch
git remote -v
```

For each repository:

1. review only its responsibility;
2. stage only its expected paths;
3. validate and commit locally;
4. inspect the new commit;
5. repeat in the next repository;
6. push in dependency order;
7. verify every final `main...origin/main` state.

Do not run `git add .` from the directory containing all four repositories,
and do not create a fifth parent repository around them.

## Troubleshooting by boundary

| Symptom | Inspect first | Likely boundary |
| --- | --- | --- |
| “not a git repository” | `Get-Location`, `git rev-parse --show-toplevel` | Wrong directory or missing clone |
| “nothing to commit” but edits are visible | Status, diff, ignore rules, exact repository | Wrong path, ignored/untracked file, or change already committed |
| Commit has wrong name/email | `git show --format=fuller` and config origins | Author metadata, not SSH |
| `Permission denied (publickey)` | `ssh -T`, `ssh-add -l`, key registration, actual SSH client | Client/account authentication |
| SSH identifies the wrong GitHub account | `ssh -vT` and offered public keys | Agent/config selecting another key |
| SSH works but push is denied | `git remote -v` and repository access | Authorization or wrong remote |
| Push rejected as non-fast-forward | Fetch, graph, left/right log | Remote contains history absent from local branch |
| Pull refuses under `--ff-only` | Graph and branch upstream | Diverged history requiring an explicit rebase/merge decision |
| File appears both staged and unstaged | Status plus both diff forms | Index and working tree contain different states |
| Changed `.gitignore` has no effect | `git ls-files` and `git check-ignore -v` | File is already tracked or another rule wins |
| Whole file changed on Windows | `git ls-files --eol` and attributes | Line-ending or formatter rewrite |
| GitHub website lacks latest edit | Local log, status, upstream, and push output | Edit uncommitted, commit unpushed, or wrong branch/repository |
| `gh` works but Git asks for credentials | Remote URL and transport | API authentication is separate from Git transport |

## Decisions recorded for this project

- Git is the local version-control engine; GitHub is the selected remote host.
- Author identity, server host verification, user authentication, repository
  authorization, and optional commit signing are separate layers.
- `CycloniteRDX` and the GitHub noreply address are the current author metadata.
- SSH remains the authoring transport; fresh public Arch clones may use HTTPS
  until they need to publish.
- The existing verified Windows key is reused. Private key material and its
  personal fingerprint remain outside the public repositories.
- GitHub's host fingerprint is checked against current official documentation
  before first acceptance on a machine.
- Outgoing Git-over-SSH does not enable `sshd` or add an inbound firewall rule.
- GitHub CLI is used only when a GitHub API/platform operation needs it.
- `main` and `origin` remain the canonical branch and remote names.
- `pull.ff only` turns divergence into an explicit decision.
- Every commit begins with status and diff inspection, stages exact paths, and
  reviews the cached diff.
- Small reviewed handbook chapters may go directly to `main`; risky,
  collaborative, experimental, or multi-commit work uses a topic branch.
- Conventional Commits describe one logical change.
- Published `main` history is corrected with new commits or reverts, not force
  pushes.
- Rescue branches and abortable operations precede history reconciliation.
- `git restore --staged` is distinguished from destructive working-tree
  restoration.
- Stashes and reflogs are temporary local recovery mechanisms, not backups.
- `.gitattributes` owns LF normalization and binary declarations across
  Windows and Arch.
- Each of the four repositories retains an independent history and clean final
  synchronization check.

## Learning and verification checklist

- [ ] I can distinguish Git, GitHub, SSH, and GitHub CLI.
- [ ] I can explain working tree, index, `HEAD`, local branch, remote, and
      remote-tracking branch.
- [ ] I know which two states `git diff` and `git diff --cached` compare.
- [ ] I can distinguish commit author metadata from SSH authentication.
- [ ] I verify GitHub's host fingerprint before accepting a first connection.
- [ ] I never copy, commit, upload, or share an SSH private key.
- [ ] I know why `ssh -T git@github.com` can report success with exit status 1.
- [ ] I can inspect configuration values together with their scope and origin.
- [ ] I check the repository root, branch, remote, status, and diff before
      staging.
- [ ] I stage exact paths and inspect the cached diff before every commit.
- [ ] I can read the two columns of `git status --short`.
- [ ] I know that a commit is local until it is pushed.
- [ ] I can explain why `git pull --ff-only` refuses divergent history.
- [ ] I fetch and inspect the graph before choosing rebase or merge.
- [ ] I create a rescue branch before rewriting unpublished local commits.
- [ ] I can unstage without discarding the working edit.
- [ ] I use revert for a published correction and reflog for a lost local
      reference.
- [ ] I know what a default stash omits and why a branch may be safer.
- [ ] I know that ignore rules do not remove already tracked content.
- [ ] I rotate a pushed credential before considering history cleanup.
- [ ] I can inspect line-ending attributes and executable-bit changes.
- [ ] I finish each affected repository at clean `main...origin/main`.

## Sources

### Git

- [Git glossary](https://git-scm.com/docs/gitglossary)
- [Pro Git: Recording changes to the repository](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)
- [`git-config(1)`](https://git-scm.com/docs/git-config)
- [`git-add(1)`](https://git-scm.com/docs/git-add)
- [`git-diff(1)`](https://git-scm.com/docs/git-diff)
- [`git-status(1)`](https://git-scm.com/docs/git-status)
- [`git-fetch(1)`](https://git-scm.com/docs/git-fetch)
- [`git-pull(1)`](https://git-scm.com/docs/git-pull)
- [`git-push(1)`](https://git-scm.com/docs/git-push)
- [`git-restore(1)`](https://git-scm.com/docs/git-restore)
- [`git-revert(1)`](https://git-scm.com/docs/git-revert)
- [`git-reflog(1)`](https://git-scm.com/docs/git-reflog)
- [`git-stash(1)`](https://git-scm.com/docs/git-stash)
- [`gitignore(5)`](https://git-scm.com/docs/gitignore)
- [`gitattributes(5)`](https://git-scm.com/docs/gitattributes)

### GitHub and conventions

- [GitHub: Checking for existing SSH keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/checking-for-existing-ssh-keys)
- [GitHub: Testing the SSH connection](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection)
- [GitHub's SSH-key fingerprints](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)
- [GitHub: Setting the commit email address](https://docs.github.com/en/account-and-profile/how-tos/email-preferences/setting-your-commit-email-address)
- [GitHub CLI: `gh auth login`](https://cli.github.com/manual/gh_auth_login)
- [GitHub CLI: `gh auth status`](https://cli.github.com/manual/gh_auth_status)
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/)

## Next guide

Guide 23 will explain printing and peripheral integration: CUPS boundaries,
local versus discovered printers, driverless IPP, legacy drivers,
NetworkManager and firewall discovery paths, polkit authorization, queues,
jobs, scanning, removable peripherals, verification, and recovery.
