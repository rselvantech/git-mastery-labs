# Demo 02 — Git Internals: The Object Model

## Overview

Most Git courses start with `git add` and `git commit`. This series
does not — and that decision is deliberate.

**Understanding Git internals is the single highest-leverage thing you can
learn about Git.** Once you see how objects are stored, hashed, and linked,
every high-level command becomes predictable. You will stop dreading error
messages because you will understand exactly what they are telling you.

In this demo we use **plumbing commands** — the low-level commands that most
developers never touch. We use them to X-ray Git and observe exactly what
happens under the hood. Every high-level command you use in Demos 06–31
is a convenience wrapper around what you build here manually.

**What this demo covers:**

- Why hashing matters and how it works — the foundation of Git's storage
- The `.git` folder — every file and folder and what it does
- The four-field structure every Git object shares
- All four object types: blob, tree, commit, annotated tag
- Creating and reading blob objects with `git hash-object` and `git cat-file`
- Why `git hash-object` and `shasum` produce different results on identical input
- Creating tree objects with `git mktree`
- The three areas: working directory, staging area, repository
- Moving files across all three areas using only plumbing commands
- Spaced recall practice (3 questions from Demo 01 — no peeking)

---

## Prerequisites

| Requirement | Detail | Verify |
|---|---|---|
| Demo 01 complete | Git installed, identity configured | `git config --global user.name` returns your name |
| Git version | 2.28.0 or newer | `git --version` |
| Terminal | macOS/Linux terminal or WSL2 on Windows | — |
| Shell basics | `cd`, `ls`, `cat`, `echo`, `mkdir`, `find`, `printf` | — |

---

## Concepts Covered

| Concept | What you will understand after this demo |
|---|---|
| Hash functions | What they are, why they matter, which algorithms exist |
| SHA-1 | How Git uses SHA-1 and why it includes a header before hashing |
| `.git` folder | Every file and subdirectory and what it does |
| Git object database | How Git stores every object as a compressed file |
| Blob | The object type Git uses to store file contents |
| Tree | The object type Git uses to store directory listings |
| Commit | The object type Git uses to store snapshots |
| Annotated tag | The object type Git uses for named release pointers |
| `git hash-object` | How to create a blob from any input |
| `git cat-file` | How to read any object from the database |
| `git mktree` | How to create a tree object manually |
| `git read-tree` | How to load a tree into the staging area |
| `git ls-files` | How to inspect the staging area |
| `git checkout-index` | How to write staged files to disk |
| Staging area / Index | The mandatory intermediate area between working dir and repo |
| Object structure | type + length + null + content — the universal format |
| Plumbing vs porcelain | Low-level vs high-level Git commands |

---

## Directory Structure

```
02-git-internals-object-model/
├── README.md                                    # this file
├── 02-git-internals-object-model-anki.csv       # Anki flashcard deck
├── 02-git-internals-object-model-quiz.md        # standalone quiz
└── src/
    ├── first-project/                           # persistent demo repo used in this and later demos
    └── break-fix/                               # isolated break-fix environment
```

> `src/first-project/` is created during the walkthrough and reused in
> Demo 03, 04, and 05. If you are starting from Demo 03 and do not have
> it, complete the walkthrough steps in this demo first.

---

## Spaced Recall

*Answer these from memory before reading anything below.
No peeking at Demo 01.*

1. What is the difference between Git and GitHub — one sentence each.
2. Name the four object types Git stores internally.
3. You run `git commit` and get an error about missing identity.
   What two config values must you set and what is the exact command for each?

<details>
<summary>Reveal answers — attempt from memory first</summary>

**1.** Git is a distributed version control system that runs locally and needs no internet to commit or read history. GitHub is a cloud hosting service for Git repositories that adds pull requests, branch protection, and CI/CD pipelines on top.

**2.** blob (file contents), tree (directory listing), commit (project snapshot), annotated tag (named release pointer).

**3.** `user.name` and `user.email`. Commands:
```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

</details>

---

## Part 0 — Why Hashing Matters

Before diving into Git objects, you need a clear mental model of what
a hash function is — because Git is built entirely on top of one.

### What is a hash function?

A hash function takes any input — a word, a file, an entire codebase —
and produces a fixed-length output called a **hash** or **digest**.

```
┌─────────────────────────────────────────────────────────────────┐
│  HASH FUNCTION PROPERTIES                                        │
│                                                                  │
│  Fixed length output                                             │
│    Any input size → always the same output length               │
│    "Hi"       → 40 characters                                   │
│    10 GB file → 40 characters  (same length)                    │
│                                                                  │
│  Deterministic                                                   │
│    Same input → always same output                              │
│    "Hello, git" → 8ab686...  (today, tomorrow, always)          │
│                                                                  │
│  One-way (irreversible)                                          │
│    Given the hash, you cannot reconstruct the input             │
│    Safe to use as a content fingerprint                          │
│                                                                  │
│  Avalanche effect                                                │
│    One character change → completely different hash             │
│    "Hello, git"  → 8ab686eafeb1f44702738c8b0f24f2567c36da6d    │
│    "hello, git"  → a8a8a8... (completely different)             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Common hash algorithms and where they are used

| Algorithm | Output length | Used in |
|---|---|---|
| MD5 | 128-bit / 32 hex chars | File checksums, legacy systems — NOT secure |
| SHA-1 | 160-bit / 40 hex chars | **Git** (object IDs), TLS certificates (deprecated) |
| SHA-256 | 256-bit / 64 hex chars | Password storage, Bitcoin, Git (future — see below) |
| SHA-512 | 512-bit / 128 hex chars | High-security cryptographic applications |

### Why Git uses SHA-1

Git uses SHA-1 to generate a unique identifier for every object it stores.

```
Any file contents
       │
       ▼
  SHA-1 hash
       │
       ▼
  40-character hex string  →  used as both the object's NAME and LOCATION
                               .git/objects/<first-2>/<remaining-38>
```

The same content always produces the same hash. This gives Git three
things for free:

```
1. DEDUPLICATION  — two files with identical contents → one hash → one
                    stored object. Git never stores the same content twice.

2. INTEGRITY      — if any byte in a stored object changes, its hash
                    changes. Git detects corruption automatically.

3. FAST LOOKUP    — finding an object by hash is a direct file lookup.
                    No index, no database query needed.
```

> **SHA-256 transition:** Git has supported `--hash=sha256` repositories
> since Git 2.29. The transition is ongoing — most repos still use SHA-1.
> SHA-1 collision attacks exist in theory but require enormous resources
> and are not a practical concern for source code repositories.

---

## Part 1 — The .git Folder

When you run `git init`, Git creates one hidden folder: `.git`.
This folder **is** the repository. Your source files are just a working
copy. Delete `.git` and all history is permanently gone.

```
first-project/
│
├── file1.txt                # working files — outside .git/
├── file2.txt                # edited, deleted, created freely here
│
└── .git/                    # THE REPOSITORY — never edit manually
    │
    ├── HEAD                 # which branch or commit you are currently on
    │                        # content: "ref: refs/heads/main"
    │                        # in detached HEAD state: the full SHA-1 hash
    │
    ├── config               # repository-level config
    │                        # overrides ~/.gitconfig for this repo only
    │
    ├── description          # human-readable repo name
    │                        # used by GitWeb only — rarely touched
    │
    ├── index                # THE STAGING AREA (binary file)
    │                        # records what will go into the next commit
    │                        # created after first git add
    │                        # inspected with: git ls-files -s
    │
    ├── objects/             # THE OBJECT DATABASE
    │   ├── info/            # metadata about object packs (starts empty)
    │   └── pack/            # packed objects (Git compresses loose objects
    │                        # here over time — git gc triggers this)
    │
    ├── refs/                # references — named pointers to commit hashes
    │   ├── heads/           # local branches: refs/heads/main
    │   └── tags/            # tags: refs/tags/v1.0.0
    │
    └── hooks/               # scripts triggered by Git events
                             # pre-commit, commit-msg, pre-push, etc.
                             # covered in Demo 25
```

### How objects/ organises files

Every object file is named by its SHA-1 hash, split 2+38:

```
SHA-1 hash:  8ab686eafeb1f44702738c8b0f24f2567c36da6d

Split:       8a  /  b686eafeb1f44702738c8b0f24f2567c36da6d
             ──     ──────────────────────────────────────
             folder  filename (38 chars)

Stored at:  .git/objects/8a/b686eafeb1f44702738c8b0f24f2567c36da6d
```

The 2+38 split limits each git repo's objects folder to a maximum of
256 subdirectories (`00` through `ff`).
Maximum folders = 16 × 16 = **256** (one per two-character hex prefix).

---

## Part 2 — The Universal Object Structure

Git stores four types of objects. Every object — regardless of type —
shares the same four-field structure:

```
┌─────────────────────────────────────────────────────────────────┐
│  UNIVERSAL GIT OBJECT STRUCTURE                                 │
│                                                                 │
│  Field 1: type      "blob"  "tree"  "commit"  "tag"             │
│  Field 2: length    size in bytes, as a decimal ASCII string    │
│  Field 3: delimiter null byte  \0                               │
│  Field 4: content   the actual data                             │
│                                                                 │
│  Example blob containing "Hello, git\n" (11 bytes):             │
│                                                                 │
│  blob SP 11 \0 Hello, git\n                                     │
│  ┬─── ┬─ ┬─ ┬  ┬──────────                                      │
│  │    │  │  │  └── content (raw bytes of the file)              │
│  │    │  │  └───── null byte delimiter  \0                      │
│  │    │  └──────── length in bytes (decimal ASCII)              │
│  │    └─────────── space separator (single space)               │
│  └──────────────── type string                                  │
│                                                                 │
│  The SHA-1 hash is computed over this ENTIRE string —           │
│  not just the content. This is why git hash-object produces     │
│  a different hash than shasum on the same raw input.            │
└─────────────────────────────────────────────────────────────────┘
```

### Git creates objects for every file and directory

Before describing each type, here is how Git maps your project to objects:

```
Your project on disk:          Git objects created:
──────────────────────         ──────────────────────────────────
Every file          →          one BLOB per unique file content
Every directory     →          one TREE per directory
Each snapshot       →          one COMMIT wrapping the root TREE
Each named release  →          one ANNOTATED TAG pointing to a COMMIT
```

This means `git add file.txt` creates a blob.
`git commit` creates a tree for each directory and a commit wrapping the root tree.

---

### BLOB — file contents

A blob (Binary Large Object) stores the raw contents of a file.

**What a blob stores:**

```
┌────────────────────────────────────────────────────────────────┐
│  BLOB OBJECT                                                    │
│                                                                 │
│  Stores:   raw byte contents of the file                       │
│            can be text, binary, images — any format            │
│                                                                 │
│  Encoding: stored as zlib-compressed binary on disk            │
│            Git uses zlib (DEFLATE algorithm) to reduce size    │
│            you cannot read blob files directly with cat        │
│            use git cat-file -p <hash> to read them             │
│                                                                 │
│  Does NOT store:                                               │
│    ✗ filename                                                   │
│    ✗ file path                                                  │
│    ✗ permissions                                               │
│    ✗ timestamps                                                 │
│                                                                 │
│  Those all live in the TREE that points to this blob           │
└────────────────────────────────────────────────────────────────┘
```

**Deduplication in action:**

```
Scenario: file1.txt and file2.txt have identical contents: "Hello, git\n"

  git add file1.txt
    → SHA-1("blob 11\0Hello, git\n") = 8ab686...
    → stores .git/objects/8a/b686...

  git add file2.txt
    → SHA-1("blob 11\0Hello, git\n") = 8ab686...  (same hash!)
    → object already exists — nothing written

  Result: 1 blob object shared by both files.
          The tree stores two entries pointing to the same blob hash.
          Storage cost for the second file = 0 bytes.
```

---

### TREE — directory listing

A tree object represents a directory. It stores a list of entries — one
per file or subdirectory — each pointing to a blob or another tree.

**Tree entry format — each line is one entry:**

```
┌──────────────────────────────────────────────────────────────────┐
│  TREE ENTRY FORMAT                                               │
│                                                                  │
│  <mode> SP <type> SP <sha1-hash> TAB <name>                      │
│   ┬────    ┬────    ┬──────────      ┬────                       │
│   │        │        │                └── filename or dirname     │
│   │        │        └─────────────────── 40-char SHA-1 hash      │
│   │        └──────────────────────────── "blob" or "tree"        │
│   └───────────────────────────────────── permission mode         │
│                                                                  │
│  IMPORTANT: the separator between <sha1-hash> and <name>         │
│             is a TAB character (\t), not a space                 │
│                                                                  │
│  Example tree contents (as shown by git cat-file -p):            │
│                                                                  │
│  100644 blob 8ab686eafeb1f44702738c8b0f24f2567c36da6d\tfile1.txt │
│  100644 blob 44aabb1234567890abcdef1234567890abcdef12\tfile2.txt │
│  040000 tree 3b95d9c4b0a5e2f6d0c1e8b2a0f3d4c5e6a7b8c9\tsrc       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Permission modes:**

```
100644  regular non-executable file     ← most common for source files
100755  executable file                 (shell scripts, binaries)
120000  symbolic link
040000  directory (another tree object)
160000  git submodule (gitlink)
```

---

### COMMIT — project snapshot

A commit is a wrapper around a tree object. It records who made a
snapshot, when, and what the full state of the project was.

**Commit object structure:**

```
┌────────────────────────────────────────────────────────────────┐
│  COMMIT OBJECT                                                 │
│                                                                │
│  tree      3b95d9...   ← SHA-1 of the ROOT tree (full state)   │
│  parent    6fb73b...   ← SHA-1 of the previous commit          │
│                          (absent on the very first commit)     │
│  author    Name <email> timestamp timezone                     │
│  committer Name <email> timestamp timezone                     │
│  \n                    ← blank line separates header from msg  │
│  commit message text                                           │
│                                                                │
│  AUTHOR vs COMMITTER:                                          │
│  Author    = person who originally wrote the change            │
│  Committer = person who applied it to this repository          │
│                                                                │
│  They differ when:                                             │
│  • A patch is emailed and applied by a maintainer              │
│    (author = patch writer, committer = maintainer)             │
│  • git rebase or git cherry-pick replays commits               │
│    (author = original writer, committer = person rebasing)     │
│  • git am applies a patch from a mailing list                  │
│                                                                │
│  For your own commits on your own machine: they are identical  │
│  Hands-on demonstration: Demo 05 (first real commits)          │
└────────────────────────────────────────────────────────────────┘
```

---

### ANNOTATED TAG — named release pointer

```
┌────────────────────────────────────────────────────────────────┐
│  ANNOTATED TAG OBJECT                                          │
│                                                                │
│  object   6fb73b...   ← SHA-1 of the commit being tagged       │
│  type     commit                                               │
│  tag      v1.0.0      ← the tag name                           │
│  tagger   Name <email> timestamp timezone                      │
│                                                                │
│  Release message text                                          │
│                                                                │
│  Distinct from a lightweight tag:                              │
│  Lightweight tag = just a file in refs/tags/ pointing to hash  │
│  Annotated tag   = a full Git object with tagger, date, msg    │
│                                                                │
│  Covered in depth in Demo 21 (Git Tags & Semantic Versioning)  │
└────────────────────────────────────────────────────────────────┘
```

---

### Loose objects vs pack files

Every object Git creates lands initially in `.git/objects/` as an
individual compressed file — one file per object. These are called
**loose objects**.

```
.git/objects/
├── 8a/b686...    ← one loose object file (a blob)
├── 3b/95d9...    ← one loose object file (a tree)
└── 6f/e073...    ← one loose object file (a commit)
```

As a repository grows — thousands of commits, large files, long history —
storing every version of every file as a separate loose object becomes
inefficient. Git solves this with **pack files**: a single binary file
that combines many objects together, with delta compression applied
between similar objects (e.g. two versions of the same file store only
the diff, not the full content twice).

```
.git/objects/pack/
├── pack-abc123.pack    ← many objects packed into one file
└── pack-abc123.idx     ← index for fast lookup inside the pack
```

Pack files are created automatically by `git gc` (garbage collection)
and when you push to or clone from a remote. You never create them manually.

> Loose objects, pack files, delta compression, and `git gc` are covered
> in depth in Demo 03.

---

## Part 3 — Plumbing vs Porcelain Commands

Git has two layers of commands:

```
PORCELAIN (high-level — what you use daily)
  git add, git commit, git push, git pull, git merge ...
  Designed for humans. Polished output. Convenience wrappers.

PLUMBING (low-level — what porcelain is built from)
  git hash-object, git cat-file, git mktree, git read-tree,
  git commit-tree, git checkout-index ...
  Designed for scripts. Raw output. Direct database access.
```

In this demo we use plumbing commands exclusively. This is not how you
work day-to-day — but doing it once manually makes every porcelain
command obvious and predictable forever after.

---

## Part 4 — Walkthrough: Initialise a Repository

All walkthrough steps in this demo run inside `src/first-project/`.
This directory is reused in Demo 03, 04, and 05.

### Step 1 — Create the project folder

```bash
cd ~/git-mastery-labs/02-git-internals-object-model/src
mkdir first-project
cd first-project
git init

# Expected:
# Initialized empty Git repository in .../src/first-project/.git/
```

### Step 2 — Explore the .git folder

```bash
ls -la .git/
# HEAD  config  description  hooks/  info/  objects/  refs/

cat .git/HEAD
# ref: refs/heads/main

# objects/ starts completely empty — no objects yet
find .git/objects -type f
# (no output)

# index does not exist yet — created after first git add
ls .git/index 2>/dev/null || echo "index does not exist yet"
# index does not exist yet
```

---

## Part 5 — Walkthrough: Creating Blob Objects

### Command introduction — git hash-object

```
NAME
  git hash-object — compute SHA-1 hash of content and optionally store it

SYNTAX
  git hash-object [options] <file>
  echo "content" | git hash-object [options] --stdin

OPTIONS
  --stdin      read content from standard input instead of a file
  -w           write the object to .git/objects/ database
               without -w: computes and prints the hash only — nothing stored
  -t <type>    specify object type (default: blob)

WHY hash-object DIFFERS FROM shasum
  shasum hashes raw content only.
  git hash-object prepends the object header before hashing:

    shasum input:          "Hello, git\n"
    git hash-object input: "blob 11\0Hello, git\n"
                            ─────────── header ──────────────────
                            (type + length + null byte + content)

  The header is the four-field object structure from Part 2.
  Same content → different hash because the inputs are different.

  Verification:
    printf "blob 11\0Hello, git\n" | shasum
    → produces the SAME hash as git hash-object --stdin
```

### Step 3 — Hash without writing (dry run)

```bash
echo "Hello, git" | git hash-object --stdin
# Output: 8ab686eafeb1f44702738c8b0f24f2567c36da6d
# Nothing written — no -w flag used

find .git/objects -type f
# (still empty — confirmed)
```

### Step 4 — Create the first blob object

```bash
echo "Hello, git" | git hash-object --stdin -w
# Output: 8ab686eafeb1f44702738c8b0f24f2567c36da6d

# Verify it was stored
find .git/objects -type f
# .git/objects/8a/b686eafeb1f44702738c8b0f24f2567c36da6d
#              ──  ──────────────────────────────────────
#              first 2 chars = folder   remaining 38 = filename

git cat-file --batch-all-objects --batch-check
# 8ab686eafeb1f44702738c8b0f24f2567c36da6d blob 11
# format: <hash> <type> <size-in-bytes>
```

What just happened inside Git:

```
INPUT:   "Hello, git\n"  (from echo — includes a trailing newline)
          │
          ▼  Git prepends the object header (see Part 2)
FULL:    "blob 11\0Hello, git\n"
          │
          ▼  SHA-1 computed over the full string
HASH:    8ab686eafeb1f44702738c8b0f24f2567c36da6d
          │
          ▼  compressed with zlib and stored
FILE:    .git/objects/8a/b686eafeb1f44702738c8b0f24f2567c36da6d
```

### Step 5 — Create a second blob from a file

```bash
# Create a temp file (using /tmp for intermediate files)
echo "Second file in git repository" > /tmp/new-file.txt

git hash-object /tmp/new-file.txt -w
# A new SHA-1 hash (different content → different hash)

# Two objects now
find .git/objects -type f
git cat-file --batch-all-objects --batch-check
# 8ab686... blob 11
# 44xxxx... blob 31
```

### Key observation — working directory is still empty

```bash
ls -la
# Only .git/ — no other files in first-project/

# The blobs live in .git/objects/ but the working directory is empty.
# Git's object database and your working directory are completely
# independent. Objects can exist in the database without appearing on disk.
```

---

## Part 6 — Walkthrough: Reading Objects with git cat-file

### Command introduction — git cat-file

```
NAME
  git cat-file — read content, type, or size of any Git object

SYNTAX
  git cat-file [option] <hash>
  git cat-file --batch-all-objects --batch-check

HASH INPUT
  Full 40-char hash: always works
  Unique prefix:     4+ characters is usually enough
                     Git rejects ambiguous prefixes

OPTIONS
  -p   pretty-print the object contents (human-readable)
  -t   print the object type: blob / tree / commit / tag
  -s   print the object size in bytes
  -e   exit code 0 if object exists, non-zero if not (use in scripts)

BATCH MODE
  --batch-all-objects --batch-check
       lists every object in the database — one line per object
       format: <hash> <type> <size-in-bytes>
       useful for auditing and debugging — no hash argument needed
```

### Step 6 — Read contents, type, and size

```bash
# Use first 4+ chars of the hash from Step 4
git cat-file -p 8ab6
# Hello, git

git cat-file -t 8ab6
# blob

git cat-file -s 8ab6
# 11
# ("Hello, git\n" = 10 chars + 1 newline = 11 bytes)
```

### Step 7 — Verify the object structure

```bash
# Reproduce the blob hash from scratch — proving the header is included:
printf "blob 11\0Hello, git\n" | shasum
# → 8ab686eafeb1f44702738c8b0f24f2567c36da6d
# Matches what git hash-object produced — the header IS part of the hash.
```

---

## Part 7 — Walkthrough: Creating a Tree Object

Blobs store file contents but not filenames. Trees store filenames by
pointing to blobs (and other trees) with a name and permissions attached.

### Command introduction — git mktree

```
NAME
  git mktree — build a tree object from a list of entries on stdin

SYNTAX
  printf "<entries>" | git mktree

INPUT FORMAT — one entry per line, fields separated as shown:

  <mode> SP <type> SP <hash> TAB <name>\n
   ┬────    ┬────    ┬──────     ┬─────
   │        │        │           └── filename or directory name
   │        │        └───────────── 40-char SHA-1 hash of the object
   │        └────────────────────── "blob" or "tree"
   └─────────────────────────────── permission mode (e.g. 100644)

  CRITICAL: the separator between <hash> and <name> MUST be a TAB (\t)
            using a space here will cause git mktree to fail or
            produce a malformed tree — use printf with \t, not echo

OUTPUT
  prints the SHA-1 hash of the newly created tree object

NOTE
  git mktree writes the tree directly to .git/objects/ — no -w flag needed
```

### Step 8 — Build the tree input manually

Before running `git mktree`, understand what input it expects.
For two blobs (file1.txt and file2.txt) the input looks like this:

```
┌─────────────────────────────────────────────────────────────────┐
│  TREE INPUT WE ARE BUILDING                                     │
│                                                                 │
│  Line 1:  100644 blob 8ab686... [TAB] file1.txt                 │
│  Line 2:  100644 blob 44aabb... [TAB] file2.txt                 │
│                                                                 │
│  Each line:                                                     │
│    100644       = regular file permission                       │
│    blob         = object type                                   │
│    8ab686...    = SHA-1 of the blob (the file's contents)       │
│    [TAB]        = mandatory tab separator                       │
│    file1.txt    = name this blob will have in the directory     │
└─────────────────────────────────────────────────────────────────┘
```

### Step 9 — Create the tree object

```bash
# Capture both blob hashes into variables
BLOB1=$(echo "Hello, git" | git hash-object --stdin)
BLOB2=$(git hash-object /tmp/new-file.txt)

echo "Blob 1: $BLOB1"
echo "Blob 2: $BLOB2"

# Build the tree — \t is the mandatory TAB before each filename
TREE_HASH=$(printf "100644 blob $BLOB1\tfile1.txt\n100644 blob $BLOB2\tfile2.txt\n" \
  | git mktree)

echo "Tree hash: $TREE_HASH"

# Verify it was created
find .git/objects -type f
git cat-file --batch-all-objects --batch-check
# 8ab686... blob 11
# 44xxxx... blob 31
# 3bxxxx... tree ...   ← new tree object
```

### Step 10 — Inspect the tree

```bash
git cat-file -p $TREE_HASH
# 100644 blob 8ab686eafeb1f44702738c8b0f24f2567c36da6d    file1.txt
# 100644 blob 44aabb...                                    file2.txt

git cat-file -t $TREE_HASH
# tree

git cat-file -s $TREE_HASH
# size in bytes of the tree object
```

The object graph after these steps:

```
  Objects in .git/objects/:        What each represents:
  ─────────────────────────────    ─────────────────────────────────────
                                    
  [TREE]  $TREE_HASH                root directory of first-project/
    │                               
    ├── 100644 blob $BLOB1 ──────►  [BLOB] "Hello, git\n"
    │          (file1.txt)          raw contents, zlib-compressed
    │                               
    └── 100644 blob $BLOB2 ──────►  [BLOB] "Second file in git repository\n"
               (file2.txt)          raw contents, zlib-compressed

  Working directory (first-project/):  STILL EMPTY
  Staging area (.git/index):           STILL EMPTY (index does not exist yet)
  Repository (.git/objects/):          3 objects — 2 blobs + 1 tree
```

Verify the working directory is still empty:

```bash
ls -la ~/git-mastery-labs/02-git-internals-object-model/src/first-project/
# only .git/ — no files on disk yet
```

---

## Part 8 — Walkthrough: Staging Area and Working Directory

Git has three areas where files live. Every file transition between
areas is explicit — nothing moves automatically.

```
┌───────────────────┐              ┌───────────────────┐              ┌───────────────────┐
│  Working          │              │  Staging Area     │              │  Git Repository   │
│  Directory        │  ─────────►  │  (Index)          │  ─────────►  │  .git/objects/    │
│                   │              │  .git/index       │              │                   │
│  files on disk    │  ◄─────────  │                   │  ◄─────────  │  blobs, trees,    │
│  you edit here    │              │  snapshot of what │              │  commits, tags    │
│                   │              │  the next commit  │              │                   │
└───────────────────┘              │  will contain     │              └───────────────────┘
                                   └───────────────────┘

   Working Dir → Staging:   git add          (porcelain)
                             git read-tree   (plumbing — used below)

   Staging → Repository:    git commit       (porcelain)
                             git commit-tree (plumbing — Demo 05)

   Repository → Staging:    git read-tree   (plumbing — used below)

   Staging → Working Dir:   git checkout-index (plumbing — used below)
                             git restore        (porcelain — Demo 08)
```

The staging area is **mandatory** in every direction — there is no direct
path from repository to working directory that bypasses it.

### Command introduction — git read-tree

```
NAME
  git read-tree — load a tree object into the staging area (index)

SYNTAX
  git read-tree <tree-hash>

WHAT IT DOES
  Reads the specified tree object from .git/objects/
  Writes entries into .git/index (creating it if needed)
  Does NOT touch the working directory — files do not appear on disk yet

WHEN TO USE
  Loading a historical tree into the staging area before checking out
  Plumbing equivalent of: git checkout <branch> (the staging half only)
```

### Command introduction — git ls-files

```
NAME
  git ls-files — inspect the contents of the staging area (index)

SYNTAX
  git ls-files [options]

OPTIONS
  (no options)   list filenames of all staged files
  -s             detailed output: permissions | hash | stage | name

STAGE NUMBER (third column with -s)
  0   normal — not in a merge conflict
  1   base (common ancestor) — appears during merge conflict
  2   ours — appears during merge conflict
  3   theirs — appears during merge conflict
```

### Command introduction — git checkout-index

```
NAME
  git checkout-index — copy files from staging area to working directory

SYNTAX
  git checkout-index [options]

OPTIONS
  -a   all files — write every file in the staging area to disk
  -f   force overwrite even if the file already exists on disk

WHAT IT DOES
  Reads entries from .git/index
  Creates the corresponding files in the working directory
  Does NOT change the staging area or repository
```

### Step 11 — Load the tree into the staging area

```bash
git read-tree $TREE_HASH

# Staging area now has two files
git ls-files
# file1.txt
# file2.txt

git ls-files -s
# 100644 8ab686... 0    file1.txt
# 100644 44aabb... 0    file2.txt
# Columns: permissions | SHA-1 hash | stage-number | filename

# Working directory is still empty
ls -la
# only .git/ — read-tree only updated the index, not the working dir
```

### Step 12 — Write staged files to working directory

```bash
git checkout-index -a

# Verify files now exist on disk
ls -l
# file1.txt
# file2.txt

cat file1.txt
# Hello, git

cat file2.txt
# Second file in git repository

# Staging area is unchanged — files are still tracked there too
git ls-files -s
# 100644 8ab686... 0    file1.txt
# 100644 44aabb... 0    file2.txt

# *****>>>>Now we have the files in all 3 places<<<<<*****
```

### Step 13 — Check git status

```bash
git status
# On branch main
#
# No commits yet
#
# Changes to be committed:
#   (use "git rm --cached <file>..." to unstage)
#         new file:   file1.txt
#         new file:   file2.txt
```

Reading this output:

```
"On branch main"
  HEAD points to refs/heads/main, which does not yet exist
  (no commits have been made — the branch has no history yet)

"No commits yet"
  The staging area has content but no commit object exists in the repo

"Changes to be committed: new file: file1.txt & file2.txt"
  Git sees these files in the staging area (.git/index)
  They are not yet committed — git commit would make them permanent
  Note: git commit has NOT been run yet in this demo
        That is the subject of Demo 05
```

### What we just did manually vs what git add + git commit do

```
What we did (plumbing):         Porcelain equivalent:
────────────────────────────    ──────────────────────────────────────────
git hash-object file -w    →    } git add file
update .git/index entry    →    }

git mktree             →    }
git commit-tree        →    } git commit -m "message"
update HEAD / branch   →    }

git read-tree          →    } git checkout <branch> (index half)
git checkout-index -a  →    } git checkout <branch> (working dir half)
```

> `git commit-tree` is the plumbing command that creates a commit object
> from a tree hash. It is demonstrated in Demo 05 when we build commits
> manually. For now: know it exists and that `git commit` calls it internally.

---

## CLI Verification Toolkit

Use these whenever something unexpected happens in any Git repo.

```bash
# ─────────────────────────────────────────────────────
# LIST ALL OBJECTS IN THE DATABASE
# ─────────────────────────────────────────────────────
find .git/objects -type f
git cat-file --batch-all-objects --batch-check

# ─────────────────────────────────────────────────────
# READ ANY OBJECT (4+ chars of hash is enough)
# ─────────────────────────────────────────────────────
git cat-file -p <hash>   # contents
git cat-file -t <hash>   # type
git cat-file -s <hash>   # size in bytes
git cat-file -e <hash>   # exists? (exit code 0 = yes)

# ─────────────────────────────────────────────────────
# CREATE OBJECTS
# ─────────────────────────────────────────────────────
echo "content" | git hash-object --stdin       # hash only — nothing stored
echo "content" | git hash-object --stdin -w    # hash + write to database
git hash-object <filename> -w                  # hash file + write

# ─────────────────────────────────────────────────────
# INSPECT THE STAGING AREA
# ─────────────────────────────────────────────────────
git ls-files          # list all tracked files
git ls-files -s       # detailed: permissions | hash | stage | name

# ─────────────────────────────────────────────────────
# MOVE FILES BETWEEN AREAS (plumbing)
# ─────────────────────────────────────────────────────
git read-tree <tree-hash>    # repository → staging area
git checkout-index -a        # staging area → working directory
```

---

## Key Takeaways

1. **`.git/` is the repository — your source files are just a working copy.** Deleting `.git/` permanently destroys all history with no recovery possible. Every other area (working directory, staging area) can be rebuilt from objects in `.git/objects/`; the reverse is not true.

2. **Every Git object is SHA-1 of (type + space + length + null byte + content) — not SHA-1 of the content alone.** The header is mandatory and part of what makes `git hash-object` produce a different result than `shasum` on identical input. Reproduce any hash independently with `printf "blob N\0content\n" | shasum`.

3. **Blobs do not store filenames — trees do.** This separation enables deduplication: two files with identical content share one blob object. The tree stores two entries pointing to the same hash. Storage cost for the duplicate is zero bytes.

4. **The staging area is mandatory — nothing goes directly from repository to working directory or back.** Every file transition passes through `.git/index`. `git add` = hash-object + update index. `git commit` = mktree + commit-tree + update HEAD. Understanding this makes every staging-related error message self-explanatory.

5. **Author and committer are different fields in a commit object.** They are identical for commits made on your own machine, but diverge during rebase (committer = person rebasing), cherry-pick, and patch application via `git am`. Both fields are permanent parts of the commit object and cannot be changed without rewriting history.

### Quick reference — commands used in this demo

| Command | What it does |
|---|---|
| `git init` | Initialise a new empty repository |
| `git hash-object --stdin` | Hash stdin — no write |
| `git hash-object --stdin -w` | Hash stdin and write to object db |
| `git hash-object <file> -w` | Hash a file and write to object db |
| `git cat-file -p <hash>` | Print object contents |
| `git cat-file -t <hash>` | Print object type |
| `git cat-file -s <hash>` | Print object size in bytes |
| `git cat-file -e <hash>` | Check object exists (exit code) |
| `git cat-file --batch-all-objects --batch-check` | List all objects with type and size |
| `git mktree` | Create a tree object from stdin |
| `git read-tree <hash>` | Load a tree into the staging area |
| `git ls-files` | List files in staging area |
| `git ls-files -s` | Detailed staging area listing |
| `git checkout-index -a` | Write all staged files to working directory |
| `find .git/objects -type f` | List all object files on disk |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `not a valid object name` | Hash prefix not long enough or typo | Use at least 4 chars; verify with `find .git/objects -type f` |
| `mktree` produces no output | TAB missing before filename | Use `printf` with `\t` — not spaces |
| `checkout-index` writes no files | Staging area empty | Run `git read-tree <hash>` first |
| `fatal: not a git repository` | `git init` not run | Run `git init` from the project root |
| Second blob has same hash as first | Contents are identical | Git deduplication working — not an error |
| `shasum` hash differs from `git hash-object` | Expected — Git hashes header + content | Use `printf "blob N\0content\n"` piped to shasum to reproduce |

---

## Break-Fix Scenario

The break-fix environment is isolated in `src/break-fix/` with its own
Git repository — separate from the `first-project/` you built during
the walkthrough. This keeps the break-fix from affecting your demo work.

All commands are run manually — this is intentional. The goal is to read
error messages cold and diagnose from them, which is exactly the skill
needed in production.

### Setup

```bash
cd ~/git-mastery-labs/02-git-internals-object-model/src/break-fix
git init
```

### The broken commands — run each one manually

```bash
# Command 1 — create and store a blob
echo "production config" | git hash-object --stdin -W

# Command 2 — read the type of that blob
HASH=$(echo "production config" | git hash-object --stdin)
git cat-file -T $HASH

# Command 3 — list all objects in the database
git cat-file --all-objects --batch-check
```

For each command:
1. Run it and read the error output carefully
2. Identify what is wrong from the error message alone
3. Fix the command
4. Re-run until it succeeds
5. Only then reveal the answers below

<details>
<summary>Reveal answers — attempt diagnosis first</summary>

**Error 1 — `-W` (uppercase W)**
The write flag is lowercase `-w`. Uppercase `-W` is not a valid option.
Git returns: `error: unknown switch 'W'`
The object is NOT written to the database.
Fix: `git hash-object --stdin -w`

**Error 2 — `git cat-file -T` (uppercase T)**
The type flag is lowercase `-t`. Uppercase `-T` is not a valid option.
Git returns: `error: unknown switch 'T'`
Note: because Error 1 means the object was never written, you must fix
Error 1 first — then Error 2 can succeed. Fix both in order.
Fix: `git cat-file -t $HASH`

**Error 3 — `--all-objects`**
The correct flag is `--batch-all-objects`. There is no `--all-objects`.
Git returns: `error: unknown option 'all-objects'`
Fix: `git cat-file --batch-all-objects --batch-check`

</details>

---

## Interview Prep

**Q1. You run `git hash-object --stdin` and `shasum` on the same input. They produce different hashes. A colleague says this means Git's hashing is broken. What is actually happening?**
Git prepends an object header before hashing: `type SP length NUL content`. For a blob containing "Hello\n" (6 bytes), Git hashes `blob 6\0Hello\n` — not `Hello\n` alone. `shasum` hashes only the raw content. The inputs differ, so the outputs differ. This is by design: the header encodes the object type and size, making it impossible to confuse a blob, tree, or commit that happen to have identical raw bytes. Verify it yourself: `printf "blob 6\0Hello\n" | shasum` produces the same hash as `git hash-object --stdin`.

**Q2. You `git add` two files with identical contents. A colleague says this wastes double the storage. Are they right?**
No. Git is content-addressed. Both `git add` calls compute the same SHA-1 hash — same content, same header, same hash — and the second call finds the object already in `.git/objects/` and writes nothing. The tree stores two entries with two different filenames both pointing to the same blob hash. Storage cost for the second file is zero bytes. This deduplication is automatic and applies to any number of identical files.

**Q3. A developer says `git add` just "copies files to a staging area." What is more precise?**
`git add` does two things: it calls `git hash-object -w` to create a blob object in `.git/objects/` (the actual content storage), and it updates `.git/index` (the staging area) with an entry mapping the filename and permissions to that blob's hash. The file is compressed with zlib and stored by hash at this point — not at commit time. `git commit` then calls `git mktree` to create tree objects from the index and `git commit-tree` to wrap the root tree in a commit object.

**Q4. You run `git read-tree <hash>` but the files do not appear in your working directory. What is happening and what is the next step?**
`git read-tree` loads the tree into the staging area (`.git/index`) only — it does not touch the working directory. This is by design: the staging area and working directory are separate areas. To write the staged files to disk, run `git checkout-index -a`. The `-a` flag writes all staged entries. This two-step plumbing sequence is what the porcelain `git checkout <branch>` does internally in one command.

---

## What's Next

**Demo 03 — SHA-1 & Object Storage Deep Dive**

```
✓ Reproduce SHA-1 hashes from scratch using the terminal — no Git
✓ Understand collision probability with real numbers
✓ Read raw compressed objects using Python's zlib library
✓ Prove the four-field object structure by matching hashes manually
✓ Learn about Git's SHA-256 transition (--hash=sha256)
✓ Understand pack files — how Git handles large repos efficiently
```

---

## References

| Topic | Resource |
|---|---|
| Git internals — official book | https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain |
| git-hash-object | https://git-scm.com/docs/git-hash-object |
| git-cat-file | https://git-scm.com/docs/git-cat-file |
| git-mktree | https://git-scm.com/docs/git-mktree |
| git-read-tree | https://git-scm.com/docs/git-read-tree |
| git-ls-files | https://git-scm.com/docs/git-ls-files |
| git-checkout-index | https://git-scm.com/docs/git-checkout-index |
| SHA-256 transition plan | https://git-scm.com/docs/hash-function-transition |

---

## Appendix — Anki Cards

**02-git-internals-object-model-anki.csv:**

```
#deck:git-mastery-labs::Section 1 - Foundations::02-git-internals-object-model
#separator:Comma
#columns:Front,Back,Tags
"After running 'echo Hello, git | git hash-object --stdin -w' twice, you check the working directory with ls -la. No files appear — only .git/. Yet git cat-file --batch-all-objects --batch-check shows two blobs. How is this possible?","Git's object database (.git/objects/) is completely independent from the working directory. git hash-object -w writes directly to the object database — it does not create any file on disk. Objects can exist in the database without any corresponding file in the working directory.","demo02 internals independence working-dir"
"You run git hash-object --stdin twice on identical input. Same or different hash?","Always the same. SHA-1 is deterministic — same content always produces the same hash. This is the foundation of Git's content-addressed storage.","demo02 internals sha1"
"You pipe identical content into git hash-object --stdin -w three times. How many blob objects are stored in .git/objects/?","One. Same content → same SHA-1 hash → same storage key. git hash-object -w writes the object only if it does not already exist. Git never stores the same content twice.","demo02 internals deduplication"
"What is the four-field structure of every Git object?","type + space + length (bytes as decimal ASCII) + null byte (\\0) + content. SHA-1 is computed over the entire string including the header — not just the content.","demo02 internals object-structure"
"Why does 'echo hello | git hash-object --stdin' produce a different hash than 'echo hello | shasum'?","Git prepends the object header ('blob 6\\0') before hashing. shasum hashes raw content only. The inputs differ so the outputs differ. Proof: printf 'blob 6\\0hello\\n' | shasum produces the same hash as git hash-object.","demo02 internals sha1 commands"
"A blob stores file contents. Where is the filename stored?","In the tree object that points to the blob. Each tree entry contains: mode | type | SHA-1 | filename. The TAB character separates the SHA-1 from the filename. Blobs have no name of their own.","demo02 internals blob tree"
"git cat-file has four single-character options. Name them and what each does.","-p: pretty-print contents. -t: print type (blob/tree/commit/tag). -s: print size in bytes. -e: exit 0 if object exists (for scripts).","demo02 commands cat-file"
"Why is .git/objects/ split into 256 subdirectories named 00 through ff?","To prevent filesystem performance problems from too many files in one directory. Git uses first 2 hex chars of SHA-1 as folder name (16×16=256 combinations) and remaining 38 chars as filename.","demo02 internals storage"
"What command lists all objects in the Git database with their hash, type, and size?","git cat-file --batch-all-objects --batch-check — prints one line per object in format: <hash> <type> <size-in-bytes>. No hash argument needed — operates on the entire database.","demo02 commands cat-file batch"
"What is the mandatory separator between the hash and filename in git mktree input?","A TAB character (\\t). Format: <mode> SP <type> SP <hash> TAB <name>. Using a space instead of TAB causes mktree to fail or produce a malformed tree.","demo02 commands mktree format"
"What command loads a tree from the repository into the staging area?","git read-tree <tree-SHA-1> — writes entries into .git/index. Does NOT touch the working directory. Files do not appear on disk until git checkout-index -a is run.","demo02 commands plumbing staging"
"What command writes all files from the staging area into the working directory?","git checkout-index -a — reads .git/index entries and creates the corresponding files on disk. The -a flag means all staged files. Does not change the staging area or repository.","demo02 commands plumbing staging"
"What is the difference between author and committer in a Git commit object?","Author = person who originally wrote the change. Committer = person who applied it to the repository. For your own commits on your own machine they are always identical. They differ in team workflows — covered in detail in Demo 05.","demo02 internals commit author-committer"
"Blobs store file contents as what format on disk?","Compressed binary — Git uses zlib (DEFLATE algorithm) to compress all objects before storing them in .git/objects/. You cannot read them with cat. Use git cat-file -p <hash> to read the decompressed contents.","demo02 internals blob compression"
"You run git add file.txt (creating a blob) but never commit. You then delete file.txt from the working directory. What is the state of the blob in .git/objects/?","The blob stays in .git/objects/ — deleting a file from the working directory never affects the object database. The blob is now unreachable (no commit or ref points to it). Git will eventually remove it when git gc runs, subject to gc.pruneExpire (default: 2 weeks) and gc.reflogExpireUnreachable (default: 30 days). Until then it is recoverable if you know the hash: git cat-file -p <hash>. Note: if the file had been committed, the blob would remain reachable through the commit chain and gc would never remove it while that commit exists. Reachability and gc configuration are covered in depth in Demo 10.","demo02 internals objects gc reachability"
```

---

## Appendix — Quiz

**02-git-internals-object-model-quiz.md:**

````
# Quiz — Demo 02: Git Internals: The Object Model

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 03.

---

**Q1.** You run `echo "hello" | git hash-object --stdin` twice in a row.
What does the second run output?

- A) A different hash — Git generates a new one each time
- B) The identical hash — same content always produces the same hash
- C) An error — the object already exists in the database
- D) Empty output — the result is cached and suppressed

<details>
<summary>Answer</summary>

**B** — SHA-1 is deterministic. Same input always → same 40-character hash.
This is the foundation of Git's deduplication: same content = same hash =
one stored object, regardless of how many times you hash it.

Trap: C is wrong — Git does not error on re-hashing. Without `-w` it just
prints the hash. With `-w` it writes only if the object does not yet exist.

</details>

---

**Q2.** You have `file1.txt` and `file2.txt` with identical contents.
You run `git add file1.txt` then `git add file2.txt`. How many blob
objects are stored in `.git/objects/`?

- A) 2 — one per file regardless of content
- B) 1 — same content, same hash, one blob shared by both
- C) 0 — blobs are only created at commit time
- D) Depends on file size

<details>
<summary>Answer</summary>

**B** — Same content → same SHA-1 → same object key → one blob stored.
Both staging area entries point to the same blob hash. Storage cost for
the duplicate file is 0 bytes.

Trap: C is wrong — `git add` creates the blob immediately in `.git/objects/`.
The commit creates tree and commit objects, not blobs.

</details>

---

**Q3.** What is the mandatory separator between the SHA-1 hash and the
filename in a `git mktree` input line?

- A) A space character
- B) A comma
- C) A TAB character (\t)
- D) A pipe character (|)

<details>
<summary>Answer</summary>

**C** — The format is: `<mode> SP <type> SP <hash> TAB <name>`.
The separator between the hash and filename must be a TAB (`\t`).
Using a space produces a malformed tree or an error.

Use `printf "100644 blob $HASH\tfilename.txt\n"` to generate correct input.

</details>

---

**Q4.** A blob object stores which of the following?

- A) File contents + filename + permissions
- B) File contents only — filename and permissions live in the tree
- C) File contents + filename (permissions stored in the tree)
- D) Filename + permissions — contents stored in a linked data block

<details>
<summary>Answer</summary>

**B** — A blob stores only the raw byte contents of a file, zlib-compressed.
The tree that points to the blob stores the filename and permissions for
that entry. This separation enables deduplication: two files with the
same contents share one blob regardless of their names.

</details>

---

**Q5.** After running `git hash-object --stdin -w` to create a blob,
you run `ls -la` in the project folder and see no new files — only `.git/`.
Yet `git cat-file --batch-all-objects --batch-check` shows the blob exists.
What explains this?

- A) git hash-object only creates a temporary blob — it disappears after the command exits
- B) The blob is stored in .git/objects/ which is independent of the working directory
- C) You need to run git add to make the blob visible
- D) The blob will appear after the next git status

<details>
<summary>Answer</summary>

**B** — Git's object database (`.git/objects/`) and the working directory
are completely independent. `git hash-object -w` writes directly to the
object database — it does not create any file in the working directory.
Objects can exist in the database with no corresponding file on disk.

This was demonstrated directly in the Demo 02 walkthrough: after creating
two blobs from stdin and a temp file, `ls -la` inside `first-project/`
showed only `.git/` — no other files.

Trap: C is wrong — git add creates blobs from working directory files.
Here the blob was created directly via plumbing — git add is not involved.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 5/5 | Import Anki CSV and move to Demo 03 |
| 4/5 | Review the wrong answer, then proceed |
| 3/5 | Re-read the relevant README section, retry quiz |
| Below 3/5 | Re-read Demo 02 and redo the walkthrough before proceeding |
````

