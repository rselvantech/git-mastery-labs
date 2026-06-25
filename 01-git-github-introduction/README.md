# Demo 01 — What is Git & GitHub

## Overview

Before writing a single Git command you need to answer two questions clearly:
**what problem does Git solve** and **what is GitHub adding on top of Git?**
Skip this and every subsequent error message fights two learning curves
simultaneously — "what does this command mean?" and "am I even using the
right tool?"

**What this demo covers:**

- The exact problem Git was built to solve — and why manual versioning fails
- What Git is, how it stores data, and why it needs no internet
- What a content-addressed object database is — the core mental model
- Key terms: repository, local repo, remote repo, and related concepts
- What GitHub adds on top of Git — and every major alternative
- Terminal recommendation for Windows — Git Bash vs WSL2
- Installing Git on macOS, Linux, and Windows
- Configuring Git identity, default branch, and editor globally
- Understanding config scopes, precedence, and real-world use cases
- Verifying a complete, working Git setup
- Spaced recall practice (3 questions from memory — no peeking)

---

## Prerequisites

| Requirement | Detail | Verify |
|---|---|---|
| OS | macOS, Linux, or Windows | — |
| Terminal | See Terminal section below before proceeding | — |
| Admin rights | Needed to install Git | — |
| No prior Git knowledge | This demo starts from zero | — |

---

## Concepts Covered

| Concept | What you will understand after this demo |
|---|---|
| Version Control | Why tracking file history matters in every project |
| Git | What it is, how it stores data, why it needs no internet |
| Content-addressed storage | How Git identifies every object by its content hash |
| Repository terms | local repo, remote repo, bare repo, working tree |
| GitHub | What it adds on top of Git |
| Git vs GitHub | Why they are completely separate things |
| SHA-1 | The hashing system that powers Git's object storage |
| git config | How Git knows who you are on every commit |
| Config scopes | system / global / local — precedence and real-world use |

---

## Directory Structure

```
01-git-github-introduction/
├── README.md                              # this file
├── 01-git-github-introduction-anki.csv   # Anki flashcard deck
└── 01-git-github-introduction-quiz.md    # standalone quiz
```

No `src/` directory for this demo — no scripts or config files required.

---

## Spaced Recall

*This is Demo 01 — no previous demo. Come back and answer these three
questions from memory before starting Demo 02. Write them down. No peeking.*

1. What is the difference between Git and GitHub — one sentence each.
2. Name the four object types Git stores internally.
3. You run `git commit` and get an error about missing identity.
   What two config values are required and what is the exact command
   to set each one?

<details>
<summary>Reveal answers — attempt from memory first</summary>

**1.** Git is a distributed version control system that runs locally on your machine and tracks the full history of your files with no internet required. GitHub is a cloud-based hosting service for Git repositories that adds collaboration features like pull requests, branch protection, and CI/CD pipelines.

**2.** blob (file contents), tree (directory listing), commit (project snapshot), annotated tag (named release pointer).

**3.** `user.name` and `user.email`. Commands:
```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

</details>

---

## Part 1 — The Problem Git Solves

### Manual versioning — the failure everyone hits

Before Git, developers saved versions of their work like this:

```
project/
├── report.docx
├── report_v1.docx
├── report_v2_FINAL.docx
├── report_v2_FINAL_reviewed.docx
└── report_ACTUALLY_FINAL_USE_THIS.docx
```

This breaks in four specific ways:

```
1. STORAGE WASTE
   Every copy duplicates the full file even if only one line changed.

2. NO CONTEXT
   You cannot see what changed between v1 and v2, or why.
   The filename tells you nothing.

3. NO COLLABORATION
   Two people editing the same file → manual merging.
   "Who changed what and when?" becomes impossible to answer.

4. NO SAFETY NET
   Delete the wrong version → gone permanently.
   No way back to a known working state.
```

### What Git replaced

Before Git, teams used centralised version control systems like SVN
and CVS. The critical difference:

```
CENTRALISED (SVN / CVS)                  DISTRIBUTED (Git)
──────────────────────────────────────   ──────────────────────────────────────
One central server holds all history.    Every developer holds the full history
Developers check out files to work.      on their own machine.

If the server goes down:                 If the server goes down:
  → nobody can commit                      → everyone keeps working
  → nobody can access history              → full history available locally

Speed: every operation hits the network  Speed: most operations are local disk
```

---

## Part 2 — What is Git?

### The definition

Git is a **distributed version control system (DVCS)**.

Plain English:

```
VERSION CONTROL  →  Git tracks every change to every file in a folder.
                    You can see the full history at any time.
                    You can travel back to any previous state.

DISTRIBUTED      →  Every developer has a complete copy of the entire
                    history on their local machine.
                    No server or internet required to commit or browse history.
```

### How Git stores data — the mental model

Git is not a file-diff tool. It is a **content-addressed object database** —
a key-value store where every object is identified by the SHA-1 hash of its
contents.

```
KEY   = SHA-1 hash (40 hexadecimal characters)
        Generated from the contents of the object
        Same content always → same hash
        Different content → different hash

VALUE = Compressed contents of the object

Example:
  Key:   8ab686eafeb1f44702738c8b0f24f2567c36da6d
  Value: (zlib-compressed) "Hello, git"
```

Every file version, every directory listing, every commit — stored as an
object in this database inside the `.git` folder. No external server needed.

### The four object types

```
┌────────────────────────────────────────────────────────────────┐
│  GIT OBJECT TYPES                                              │
│                                                                │
│  BLOB          →  A file (any file, any content type)         │
│  TREE          →  A directory listing (contains blobs/trees)  │
│  COMMIT        →  A snapshot of the project + metadata        │
│  ANNOTATED TAG →  A named, permanent pointer to a commit      │
│                   (used for releases and versioning)          │
└────────────────────────────────────────────────────────────────┘
```

Demos 02–05 cover all four in complete depth.

### Content-addressed object database

This is the most important mental model in Git. Everything else follows
from it.

A **content-addressed store** means objects are identified and located by
their content — not by a filename or a location. Git computes a SHA-1 hash
of an object's contents and uses that hash as both the identifier and the
storage key.

```
Traditional file storage:        Content-addressed storage (Git):
────────────────────────────     ────────────────────────────────────
Location determines identity.    Content determines identity.
"The file is at /src/app.js"     "The file with hash 8ab686... "
Move it → same content,           Move or rename it → same hash,
  different path, different name    same identity, no duplication

Consequence:                     Consequence:
  Two files with same content      Two files with same content
  are stored twice                 share ONE stored object — automatic
                                   deduplication, zero extra cost
```

Practical implications you will see throughout this series:

```
→ git add is fast for unchanged files — hash matches, object already exists
→ Renaming a file costs nothing in Git — the blob hash does not change
→ History is tamper-evident — changing any object changes its hash,
  which breaks the chain of references pointing to it
→ git reflog and recovery work because objects persist until gc runs
```

Git's objects are stored as files in `.git/objects/`, named by their hash.
Four types exist: blob (file), tree (directory), commit (snapshot),
annotated tag (named release pointer). Covered in depth in Demo 02.

### What Git does NOT need

```
✗ Internet connection  — everything works locally
✗ A server             — your laptop is the server
✗ GitHub               — Git existed 7 years before GitHub
✗ Any external service — Git is entirely self-contained
```

### A brief history

Linus Torvalds created Git in **April 2005** after the Linux kernel
project lost access to BitKeeper. He wrote the first working version
in 10 days. Design goals: speed, data integrity, distributed non-linear
workflows. It is now used in virtually every software project on Earth.

---

## Part 3 — Key Repository Terms

These terms appear constantly. Understand them now before they appear
mid-demo without explanation.

```
┌─────────────────────────────────────────────────────────────────────┐
│  REPOSITORY (repo)                                                  │
│  The database where Git stores all versions of all files.           │
│  Physically: the hidden .git/ folder inside your project.           │
│  "Initialise a repo" = run git init, which creates .git/            │
│                                                                     │
│  LOCAL REPOSITORY                                                   │
│  The repo on your own machine.                                      │
│  You commit to it, branch in it, view history from it —             │
│  all without any network connection.                                │
│                                                                     │
│  REMOTE REPOSITORY                                                  │
│  A copy of the repo hosted somewhere else — GitHub, GitLab,         │
│  a colleague's machine, a server.                                   │
│  You push your local commits to it. Others pull from it.            │
│  "origin" is the conventional name for your primary remote.         │
│                                                                     │
│  WORKING TREE (working directory)                                   │
│  The actual files you see and edit on disk — outside .git/          │
│  Git tracks changes between the working tree and the repository.    │
│                                                                     │
│  CLONE                                                              │
│  A full copy of a remote repository, downloaded to your machine.    │
│  Includes the entire history, all branches, all objects.            │
│  git clone creates both the local repo and the working tree.        │
│                                                                     │
│  BARE REPOSITORY                                                    │
│  A repo with no working tree — just the .git/ contents.            │
│  Used on servers (GitHub's storage is bare repos internally).       │
│  You never edit files directly in a bare repo.                      │
│  Created with: git init --bare                                      │
└─────────────────────────────────────────────────────────────────────┘
```

Relationship between them:

```
Your machine (local)              github.com (remote)
─────────────────────             ───────────────────
working tree                      bare repository
    │  ▲                               │  ▲
    ▼  │                               ▼  │
local repository  ── push ──►  remote repository
                  ◄── pull ──
```

---

## Part 4 — What is GitHub?

GitHub is a **cloud-based hosting service for Git repositories**.
It is not Git. It uses Git.

```
┌──────────────────────────────────────────────────────────────────┐
│  WHAT GITHUB ADDS ON TOP OF GIT                                  │
│                                                                  │
│  Remote storage    →  Your repo in the cloud, accessible         │
│                       from any machine, acts as backup           │
│                                                                  │
│  Collaboration     →  Multiple developers push and pull          │
│                       to the same remote repository              │
│                                                                  │
│  Pull Requests     →  Propose, discuss, review, and approve      │
│                       changes before merging to main             │
│                                                                  │
│  GitHub Actions    →  CI/CD pipelines triggered by push/PR       │
│                                                                  │
│  Branch protection →  Require reviews and passing tests          │
│                       before a branch can be merged              │
│                                                                  │
│  Secret scanning   →  Alert if credentials are accidentally      │
│  + Dependabot         committed or dependencies have CVEs        │
└──────────────────────────────────────────────────────────────────┘
```

### GitHub alternatives

| Platform | Common use case |
|---|---|
| **GitHub** (Microsoft) | Open source + enterprise — most widely used |
| **GitLab** | Strong built-in CI/CD, self-hosted option |
| **Bitbucket** (Atlassian) | Jira-integrated shops |
| **Azure DevOps** (Microsoft) | Enterprise Microsoft/Azure environments |
| **Gitea** | Lightweight, fully open source self-hosted |

This series uses GitHub. All Git concepts apply identically on every platform.

### Git vs GitHub — full comparison

```
┌───────────────────────┬──────────────────────────┬──────────────────────────┐
│ Feature               │ Git                       │ GitHub                   │
├───────────────────────┼──────────────────────────┼──────────────────────────┤
│ What it is            │ VCS software              │ Cloud hosting service    │
│ Created               │ Linus Torvalds, 2005      │ Founded 2008, owned by MS│
│ Where it lives        │ Your local machine        │ github.com               │
│ Internet required?    │ No                        │ Yes (for remote ops)     │
│ Free?                 │ Yes, always               │ Free tier + paid plans   │
│ Open source?          │ Yes — GPL-2.0             │ No — proprietary         │
│ Pull Requests?        │ No                        │ Yes                      │
│ CI/CD pipelines?      │ No                        │ Yes — GitHub Actions     │
│ Works without GitHub? │ Yes — completely          │ N/A — needs Git first    │
└───────────────────────┴──────────────────────────┴──────────────────────────┘

KEY INSIGHT:
  Git is the engine. GitHub is the garage.
  You can drive without the garage.
  The garage makes it much easier to park, share, and service the car.
```

---

## Part 5 — Terminal Setup (Windows)

> Skip this section if you are on macOS or Linux.

### Git Bash vs WSL2 — professional comparison

| Dimension | Git Bash | WSL2 |
|---|---|---|
| What it is | Minimal bash port bundled with Git for Windows | Full Linux kernel running inside Windows |
| Installation | Bundled with Git installer — zero extra steps | Requires one-time WSL2 setup |
| Linux tooling | Partial — core utils only, some missing | Complete — full `apt` ecosystem |
| Performance | Slower on I/O-heavy operations | Near-native Linux speed |
| Docker integration | Separate Docker Desktop install needed | Native Docker Engine inside WSL2 |
| VS Code integration | Works — opens Windows-side files | VS Code Remote-WSL — best experience |
| kubectl, helm, aws cli | Installable but can behave differently | Install natively — behaves exactly as Linux docs describe |
| File storage | Windows filesystem (C:\) | Linux filesystem (`/home/`) — faster Git operations |
| Recommended for | Occasional Git-only tasks | DevOps, Kubernetes, Terraform, Docker — any serious tooling |

### Recommendation

**Use WSL2.** For DevOps work — Git, Docker, Kubernetes, Terraform,
AWS CLI — WSL2 is the right choice. Every tool behaves exactly as the
Linux documentation describes. No translation layer, no path quirks,
no missing utilities.

Your setup (repos in WSL2 Ubuntu, VS Code via Remote-WSL) is already
the correct professional configuration. Stick with it.

```bash
# Your setup — confirmed correct:
# Repos live at:     /home/yourname/  (WSL2 Ubuntu filesystem)
# VS Code connects:  Remote-WSL extension → "Open Folder in WSL"
# Terminal:          WSL2 Ubuntu terminal (or Windows Terminal → Ubuntu tab)
```

> **Windows Terminal** (free, from Microsoft Store) is the recommended
> terminal app on Windows. It runs WSL2, PowerShell, and CMD in tabs.
> Much better experience than the default Ubuntu window.

---

## Part 6 — Installing Git

### macOS

```bash
# Homebrew (recommended)
brew install git

# Verify
git --version
# Expected: git version 2.x.x
```

### Linux — Debian / Ubuntu (including WSL2)

```bash
sudo apt update && sudo apt install git -y
git --version
```

### Linux — Fedora / RHEL

```bash
sudo dnf install git -y
git --version
```

### Windows (Git Bash only — if not using WSL2)

Download from https://git-scm.com/download/win → run installer → accept
defaults → open Git Bash from Start menu.

### Version check

The latest Git for Windows release is 2.54.0 (April 2026).
The latest stable Git release is 2.53.0 (February 2026).

**Minimum version for this series: Git 2.28.0**

`git switch` and `git restore` (Demo 08) require 2.23+.
`git switch -c` with `--orphan` improvements require 2.28+.

```bash
git --version
# If below 2.28.0:

# Ubuntu/WSL2:
sudo add-apt-repository ppa:git-core/ppa
sudo apt update && sudo apt install git

# macOS:
brew upgrade git
```

---

## Part 7 — Configuring Git

Git requires your name and email before you can make your first commit.
These are attached to every commit you create — permanently, as part of
the commit object stored in the database.

### The three configuration scopes

```
┌────────────┬───────────────────────────────┬──────────────────────────────┐
│ Scope      │ File location                  │ Applies to                   │
├────────────┼───────────────────────────────┼──────────────────────────────┤
│ system     │ /etc/gitconfig                 │ All users on this machine    │
│ global     │ ~/.gitconfig                   │ All repos for your user      │
│ local      │ .git/config (inside repo)      │ This repo only               │
└────────────┴───────────────────────────────┴──────────────────────────────┘

PRECEDENCE: local wins over global, global wins over system.
Most specific scope always takes priority.
```

### Real-world use cases for each scope

```
SYSTEM — set by a company IT/ops team on managed machines
  git config --system core.autocrlf false
  → Enforced for all users on a shared build server.
  You rarely touch this as a developer.

GLOBAL — your personal identity and preferences
  git config --global user.name "Your Name"
  git config --global user.email "your@email.com"
  git config --global core.editor "code --wait"
  → Set once. Applies to every repo you ever create or clone.

LOCAL — per-project overrides
  git config --local user.email "work@company.com"
  → Override your personal email just for one client project.
  → Common when you contribute to open source using a personal email
    but use a company email at work. Same machine, different repos,
    different identities.
```

### Precedence in action — a concrete example

```bash
# ~/.gitconfig (global):
#   user.email = personal@gmail.com

# Inside ~/work-project/.git/config (local):
#   user.email = you@company.com

cd ~/work-project
git config user.email
# → you@company.com   (local overrides global)

cd ~/personal-project
git config user.email
# → personal@gmail.com  (no local override, global applies)
```

### Set your global identity

```bash
# Name on all commits
git config --global user.name "Your Name"

# Email — explained in detail below
git config --global user.email "your@email.com"

# Default branch name — 'main' is the modern standard
# Before Git 2.28 this defaulted to 'master'
git config --global init.defaultBranch main

# Editor for commit messages — explained below
git config --global core.editor "code --wait"
```

### What is core.editor and what does "code --wait" mean?

`core.editor` is the text editor Git opens when it needs you to write
or edit text interactively. This happens in three situations:

```
1. git commit (without -m flag)
   → Opens editor for you to write a multi-line commit message
   → You save and close → Git reads the file → creates the commit

2. git rebase -i HEAD~3
   → Opens editor showing list of commits to rebase
   → You edit the action words (pick/squash/reword)
   → Save and close → rebase proceeds

3. git merge (when auto-merge message needs editing)
   → Opens editor with the default merge commit message
   → Edit if needed, save and close
```

`"code --wait"` means:

```
code         → launch VS Code
--wait       → Git waits here until you close the VS Code tab/file
               Without --wait: Git continues immediately before
               you have finished writing — commit message is empty

# What happens step by step:
git commit
  → Git opens a new VS Code tab with a temporary file (COMMIT_EDITMSG)
  → You write the commit message
  → You close the tab (not VS Code, just the tab)
  → VS Code signals to Git: "file closed, done"
  → Git reads the message, creates the commit
```

If you prefer to stay in the terminal:

```bash
git config --global core.editor "nano"   # beginner-friendly
git config --global core.editor "vim"    # if you know vim
```

### GitHub email — primary vs secondary

GitHub allows you to have one **primary** email and multiple
**secondary (backup)** emails on your account.

**Both work for commit linking.** GitHub links a commit to your account
if the commit's author email matches *any* email associated with your
account — primary or secondary.

```bash
# Check which emails are on your GitHub account:
# GitHub → Settings → Emails

# Practical setup:
git config --global user.email "primary@email.com"
# OR any secondary email on your GitHub account
# Both will correctly link commits to your profile
```

One common pattern: use your personal primary email globally, and
override with a work email locally for work repositories:

```bash
cd ~/work-project
git config --local user.email "you@company.com"
# Add this work email as a secondary on GitHub → commits link correctly
```

---

## CLI Verification

```bash
# ─────────────────────────────────────────────────────
# CHECK 1: Git version
# ─────────────────────────────────────────────────────
git --version
# Must be 2.28.0 or newer

# ─────────────────────────────────────────────────────
# CHECK 2: Identity is configured
# ─────────────────────────────────────────────────────
git config --global user.name
# Must return your name — not empty

git config --global user.email
# Must return your email — not empty

# ─────────────────────────────────────────────────────
# CHECK 3: Review your full global config
# ─────────────────────────────────────────────────────
git config --global --list
# Shows only keys that YOU have set globally.
# Empty output means nothing is configured yet.

# ─────────────────────────────────────────────────────
# CHECK 4: Full config with source file
# ─────────────────────────────────────────────────────
git config --list --show-origin
# Shows every active config value AND which file it comes from.
# Example output inside a cloned repo:
#
# file:/home/you/.gitconfig    user.name=yourname
# file:/home/you/.gitconfig    user.email=you@email.com
# file:/home/you/.gitconfig    init.defaultbranch=main
# file:.git/config             core.repositoryformatversion=0
# file:.git/config             remote.origin.url=https://github.com/...
# file:.git/config             branch.main.remote=origin
#
# Reading this output:
#   ~/.gitconfig entries     = your global settings
#   .git/config entries      = settings specific to this repo
#   /etc/gitconfig entries   = system-wide (rarely seen)
```

### What is core.pager?

You may see `core.pager=` in your config list (with an empty value):

```bash
git config --global --list
# core.pager=       ← empty value
```

`core.pager` controls which pager program Git uses to display long
output (like `git log` or `git diff`) that does not fit on one screen.
Default is `less`. Setting it empty (`core.pager=`) disables paging —
Git prints output directly to the terminal without pausing.

```bash
# Disable paging (output flows straight through — no "press q to exit"):
git config --global core.pager ""

# Restore the default (less):
git config --global core.pager "less"

# OR unset it entirely (falls back to built-in default):
git config --global --unset core.pager
```

Common reason for an empty `core.pager`: set deliberately to avoid
the pager in scripts or when piping output to another command.

---

## Key Takeaways

1. **Git and GitHub are separate tools with different purposes.** Git is a local VCS that works with no internet — it was created in 2005 and GitHub did not exist until 2008. Treating them as the same thing leads to confusion about which errors come from Git and which come from GitHub.

2. **Git is a content-addressed object database, not a file-diff tool.** Every object is identified by the SHA-1 hash of its contents — same content always produces the same hash, and the same object is never stored twice. This gives Git automatic deduplication, tamper-evident history, and fast lookups.

3. **Config scope precedence is local > global > system — most specific always wins.** Use global for your identity and defaults; use local to override per repository (e.g. a work email for one project). Forgetting this causes commits to appear under the wrong identity on GitHub.

4. **`core.editor "code --wait"` — the `--wait` flag is not optional.** Without it, Git continues immediately after launching VS Code, reads an empty message file, and creates an empty commit. Every Git editor integration requires a blocking flag equivalent to `--wait`.

5. **GitHub links commits by email — primary and secondary emails both work.** If your commit author email does not match any email on your GitHub account, commits show as an unknown user. Add the email under GitHub Settings → Emails rather than changing your git config.

### Quick reference — commands used in this demo

| Command | What it does |
|---|---|
| `git --version` | Check Git version |
| `git config --global user.name "Name"` | Set global author name |
| `git config --global user.email "email"` | Set global author email |
| `git config --global init.defaultBranch main` | Set default branch name |
| `git config --global core.editor "code --wait"` | Set commit editor |
| `git config --global core.pager ""` | Disable output paging |
| `git config --global --list` | List all global settings |
| `git config --list --show-origin` | All settings with source file |
| `git config --global --unset <key>` | Remove a global config key |
| `git config --local user.email "email"` | Override email for one repo |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `git: command not found` | Git not installed or not in PATH | Follow install steps for your OS |
| `git version 2.17` or older | Version too old for all demos | Upgrade — see install section |
| `git config user.name` returns nothing | Identity not configured | Run `git config --global user.name "Name"` |
| `git commit` opens wrong editor | core.editor not set or wrong path | `git config --global core.editor "code --wait"` |
| Commits show as unknown user on GitHub | Email mismatch | Add local git email to GitHub account under Settings → Emails |
| `git config --list` shows no entries | Global config file missing or empty | Run all `git config --global` commands from Part 7 |

---

## Break-Fix Scenario

> **Warning — this scenario modifies your real global Git config.**
> Before running anything, save your current values:

```bash
# Save current values — paste output somewhere safe
git config --global --list
```

Now run the broken sequence:

```bash
git config --Global user.name "TestUser"
git config --global user.Email "test@example.com"
git config --global init.defaultBranch Production
```

Without looking at the answers:
1. Which command(s) failed and with what error?
2. What is wrong with each?
3. What does this mean for your actual config — was it changed or not?

**After diagnosing — restore your correct values immediately:**

```bash
# Replace with YOUR actual values from the saved output above
git config --global user.name "Your Actual Name"
git config --global user.email "your.actual@email.com"
git config --global init.defaultBranch main

# Verify restored correctly
git config --global --list
```

<details>
<summary>Reveal answers — restore your config before opening this</summary>

**Error 1 — `--Global` (capital G)**
Git flags are case-sensitive. `--Global` is not a recognised option.
Git returns: `error: unknown switch 'G'`
The command fails entirely — `user.name` is NOT changed.

**Error 2 — `user.Email` (capital E)**
Config keys are case-insensitive in Git internally, so this actually
works — Git normalises it to `user.email`. No error is produced.
However it is non-standard convention — always use lowercase keys.

**Error 3 — `init.defaultBranch Production` (capital P)**
Branch names are case-sensitive. `Production` and `production` are
different branch names. No error — Git accepts it. But any new repo
you initialise will have its default branch named `Production`, which
will not match GitHub's `main` and will cause a naming mismatch
on first push.
Fix: `git config --global init.defaultBranch main`

**Net result on your config:**
- `user.name` — unchanged (Error 1 failed)
- `user.email` — changed to `test@example.com` (Error 2 worked)
- `init.defaultBranch` — changed to `Production` (Error 3 worked with wrong value)

This is why restoring your config after this scenario is important.

</details>

---

## Interview Prep

**Q1. A colleague says their commits show as "unknown user" on GitHub even though `git config --global user.name` and `user.email` are set. What is the most likely cause and how do you fix it?**
The email in `git config` does not match any email associated with their GitHub account. GitHub links commits by comparing the commit author email against all emails on the account — primary and secondary. The fix is to add the local git email to GitHub under Settings → Emails, or to change `user.email` to one already on the account. Running `git config --global user.email` reveals the current value; running `git config --list --show-origin` shows which config file it comes from — useful when a local scope override is silently winning.

**Q2. You have a personal email set globally and a work email needed for one project. How do you configure this without breaking commits in your personal repos?**
Set the work email with local scope inside that project only: `git config --local user.email "you@company.com"`. This writes to `.git/config` inside the repo and overrides the global value for that repo only. All other repos continue using the global personal email. Verify with `git config user.email` from inside the project — it should show the work email — and from outside — it should show the personal email.

**Q3. A developer sets `core.editor "code"` without `--wait` and runs `git commit`. What happens and why?**
Git launches VS Code and immediately continues without waiting for the editor to close. It reads the temporary commit message file (`COMMIT_EDITMSG`) before the developer has written anything — the file is empty — and either creates an empty commit or aborts with an error about an empty commit message, depending on `commit.cleanup` config. Fix: `git config --global core.editor "code --wait"`. The `--wait` flag tells VS Code to hold the terminal process open until the tab is closed, signalling to Git that the message is ready.

**Q4. Someone asks you to explain Git's config scope precedence in a production context — where does this actually matter?**
In a shared build server or CI environment, a system-level config (`/etc/gitconfig`) may enforce policies like `core.autocrlf false` or a specific credential helper. A developer's global config overrides these for their user. A local repo config overrides the global. The precedence — local wins over global wins over system — means project-specific settings are always portable and cannot be overridden by machine-level config. A common production scenario: a company's CI runner has system-level identity config that is correctly overridden by the pipeline's local repo config for the deployment identity.

---

## What's Next

**Demo 02 — Git Internals: The Object Model**

```
✓ Initialise a real Git repository and explore every file in .git/
✓ Create blob objects manually with git hash-object
✓ Read any object back with git cat-file
✓ Create a tree object with git mktree
✓ Load a tree into the staging area with git read-tree
✓ Write staged files to disk with git checkout-index
✓ Understand exactly what git add does internally
```

---

## References

| Topic | Resource |
|---|---|
| Official Git documentation | https://git-scm.com/doc |
| Pro Git book (free) | https://git-scm.com/book/en/v2 |
| git-config docs | https://git-scm.com/docs/git-config |
| GitHub documentation | https://docs.github.com |
| Git history — Linus Torvalds talk | https://www.youtube.com/watch?v=4XpnKHJAok8 |
| WSL2 setup guide | https://learn.microsoft.com/en-us/windows/wsl/install |

---

## Appendix — Anki Cards

**01-git-github-introduction-anki.csv:**

```
#deck:git-mastery-labs::Section 1 - Git Foundations & Internals::01-git-github-introduction
#separator:Comma
#columns:Front,Back,Tags
"You are on a flight with no internet. You need to commit 3 changes and read 50 commits of history. Can Git do this — and why?","Yes — completely. Git stores every commit, every file version, and all history inside .git/ on your local machine. Internet is only required to push to or pull from a remote like GitHub. Both operations are fully local.","demo01 fundamentals offline distributed"
"A colleague says they pushed code to Git. What is imprecise about this statement?","You push to a remote repository (GitHub / GitLab / origin) — not to Git itself. Git is the version control tool running locally. The correct statement: pushed to GitHub (or pushed to origin).","demo01 fundamentals git-vs-github"
"Git uses content-addressed storage. You have 10 files with identical contents. How many objects will Git store for those 10 files?","One. Git computes a SHA-1 hash of each object's contents. Same content → same hash → same storage key → one stored object. This is called deduplication and is automatic. (git add is covered in Demo 04 — this card tests the storage concept only.)","demo01 internals content-addressed deduplication"
"What does content-addressed storage mean in Git?","Objects are identified by what they contain, not where they are stored. Git computes SHA-1 hash of each object's contents and uses that hash as the storage key. Move or rename a file — the hash and stored object do not change.","demo01 internals content-addressed"
"Name the three Git config scopes in order from lowest to highest precedence.","system (lowest) < global < local (highest). Local always wins. global overrides system. Most specific scope takes priority.","demo01 config scopes precedence"
"You use personal@gmail.com globally but need work@corp.com for one project. What is the exact command to set the override?","cd into the project, then: git config --local user.email work@corp.com — this writes to .git/config inside the repo and overrides the global value only for that repository.","demo01 config local override"
"git commit opens VS Code but creates an empty commit before you finish writing. What is wrong and what is the fix?","core.editor is set to 'code' without --wait. Without --wait, Git continues immediately rather than waiting for you to close the file. Fix: git config --global core.editor 'code --wait'","demo01 config editor core.editor"
"What is core.editor and when does Git open it during a git commit?","core.editor is the text editor Git opens when you run git commit without the -m flag. Git writes a temporary file (COMMIT_EDITMSG), opens it in the editor, waits for you to save and close, then reads the message. Other uses (rebase, merge) are covered in later demos.","demo01 config core.editor usage"
"Your commits appear as unknown user on GitHub. user.name and user.email are set locally. What is the most likely cause?","The email in git config does not match any email on your GitHub account. GitHub links commits by matching the commit author email against all emails (primary + secondary) on your account. Add the local email to GitHub under Settings → Emails.","demo01 config github email"
"What does core.pager do, and what does an empty value (core.pager=) mean?","core.pager sets the program Git uses to page long output (default: less). An empty value disables paging — output flows directly to the terminal without stopping. Set with: git config --global core.pager '' to disable.","demo01 config core.pager"
"What command shows every active Git config value AND the file each value comes from?","git config --list --show-origin — prints source file path alongside each key=value pair. Essential for debugging when the same key appears in multiple scopes.","demo01 config commands"
"What is the difference between a local repository, a remote repository, and a bare repository?","Local repo: the .git/ folder on your machine — you commit and branch here. Remote repo: a copy hosted elsewhere (GitHub) — you push to and pull from it. Bare repo: a repo with no working tree — just the .git/ contents — used on servers; you never edit files directly in it.","demo01 fundamentals repo-types"
```

---

## Appendix — Quiz

**01-git-github-introduction-quiz.md:**

````
# Quiz — Demo 01: What is Git & GitHub

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 02.

---

**Q1.** You are on a flight with no internet. You need to make commits
and browse the last 100 commits of history. Can you do this?

- A) No — Git requires internet to commit
- B) No — Git requires internet to read history
- C) Yes — both operations are fully local
- D) Yes for commits, No for history

<details>
<summary>Answer</summary>

**C** — All commit and history operations are local. Git stores the entire
history inside `.git/` on your machine. Internet is only required for
push, pull, fetch — operations that communicate with a remote.

Trap: D is wrong — history (`git log`) is also fully local. No network needed.

</details>

---

**Q2.** A colleague says "push your changes to Git." What is imprecise?

- A) Nothing — it is correct
- B) You push to a remote (GitHub/GitLab/origin) — not "to Git"
- C) Push is not a valid Git operation
- D) You cannot push without a pull request first

<details>
<summary>Answer</summary>

**B** — You push to a remote repository (GitHub, GitLab, origin).
Git is the version control tool running locally — you do not push "to Git".
The remote is what receives the push.

</details>

---

**Q3.** You have two files with completely identical contents.
`git add` both. How many blob objects are stored in `.git/objects/`?

- A) 2 — one per file
- B) 1 — same content, same hash, one object
- C) 0 — blobs are only stored at commit time
- D) Depends on file size

<details>
<summary>Answer</summary>

**B** — Git is content-addressed. Same content → same SHA-1 hash → same
storage key → one object stored, shared by both files. Deduplication is
automatic and free regardless of file size.

Trap: C is wrong — `git add` creates blob objects immediately, not at commit time.

</details>

---

**Q4.** Your global config has `user.email = personal@gmail.com`.
Inside `~/work-project/.git/config` you set `user.email = you@corp.com`.
Which email appears on commits made inside `~/work-project`?

- A) personal@gmail.com — global always wins
- B) you@corp.com — local scope overrides global
- C) Git uses both and picks the most recent
- D) Git throws an error on conflict

<details>
<summary>Answer</summary>

**B** — Config scope precedence: local wins over global wins over system.
The local `.git/config` value always overrides the global `~/.gitconfig`
value for the same key, for that repo only.

</details>

---

**Q5.** You run `git commit` without `-m`. VS Code opens but Git
creates an empty commit before you finish typing. What is wrong?

- A) You need to use `-m` flag — no editor mode exists
- B) `core.editor` is set to `"code"` without `--wait`
- C) VS Code is not supported as a Git editor
- D) You need to save the file before VS Code opens

<details>
<summary>Answer</summary>

**B** — Without `--wait`, Git launches VS Code and immediately continues
without waiting for you to write the message. It reads an empty file and
creates an empty commit. Fix: `git config --global core.editor "code --wait"`

</details>

---

Score guide:

| Score | Action |
|---|---|
| 5/5 | Import Anki CSV and move to Demo 02 |
| 4/5 | Review wrong answers, then proceed |
| 3/5 | Re-read relevant README sections, retry quiz |
| Below 3/5 | Re-read Demo 01 before proceeding |
````