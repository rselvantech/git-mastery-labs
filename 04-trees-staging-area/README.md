# Demo 04 — Trees, Staging Area & Working Directory

## Overview

In Demo 02 you created a flat tree with two blobs using plumbing commands.
Real projects are not flat — they have nested directories, subdirectories,
and files at multiple levels. A Git tree object can point to other tree
objects, creating a hierarchy that mirrors your entire project structure.

In this demo you build that nested structure manually using plumbing
commands, then switch to the porcelain `git add` command and observe
exactly what it does to the index. By the end you will understand the
staging area deeply enough to use `git add` with complete confidence and
to diagnose any staging area problem you encounter.

**What this demo covers:**

- Nested tree objects — trees pointing to trees
- How Git represents a full directory hierarchy as linked objects
- `git add` — what it does to `.git/index` step by step
- `git rm --cached` — removing files from the index without touching disk
- `git ls-files` in depth — every useful option
- `git diff --cached` — comparing staging area against last commit
- `git diff` — comparing working directory against staging area
- The index binary format — what `.git/index` actually contains
- Spaced recall practice (3 questions from Demo 03 — no peeking)

---

## Prerequisites

| Requirement | Detail | Verify |
|---|---|---|
| Demos 02 and 03 complete | `src/first-project/` exists with commits | `git log --oneline` inside `src/first-project/` |
| Git version | 2.28.0 or newer | `git --version` |
| Terminal | macOS/Linux or WSL2 | — |

---

## Concepts Covered

| Concept | What you will understand after this demo |
|---|---|
| Nested trees | How Git represents subdirectories as tree objects pointing to trees |
| Root tree | The top-level tree that a commit points to |
| `git add` internals | Exactly what happens in `.git/index` on every add |
| `git rm --cached` | Remove from staging area without deleting from disk |
| `git ls-files` options | All useful flags for inspecting staged content |
| `git diff --cached` | What will change when you commit |
| `git diff` | What has changed since the last `git add` |
| Index binary format | What `.git/index` stores and how to inspect it |
| Staging area states | Untracked, tracked, modified, staged, deleted |

---

## Directory Structure

```
04-trees-staging-area/
├── README.md                          # this file
├── 04-trees-staging-area-anki.csv     # Anki flashcard deck
├── 04-trees-staging-area-quiz.md      # standalone quiz
└── src/
    ├── first-project/                 # continued from Demo 02 and 03
    └── break-fix/                     # isolated break-fix environment
```

```
# .gitignore entries for this demo
src/first-project/
src/break-fix/
```

> `src/first-project/` continues from Demo 03.
> If you do not have it, complete the Demo 02 walkthrough first.

---

## Spaced Recall

*Answer from memory before reading anything below. No peeking at Demo 03.*

1. What is the difference between a loose object and a pack file?
2. You run `git gc`. Are unreachable objects deleted immediately? Why?
3. You want to reproduce the SHA-1 hash Git produces for a file
   containing "hello\n". What exact command do you run?

<details>
<summary>Reveal answers — attempt from memory first</summary>

**1.** A loose object is an individual zlib-compressed file stored at `.git/objects/<first2>/<remaining38>` — one file per object, created immediately on `git hash-object -w` or `git add`. A pack file combines many objects into a single binary file using delta compression. Pack files are created by `git gc` or during push/clone operations and are significantly more storage-efficient for repos with many similar object versions.

**2.** No — not by default. `git gc` removes unreachable loose objects only after both expiry windows pass: `gc.pruneExpire` (default: 2 weeks) is the minimum age before a loose unreachable object is pruned, and `gc.reflogExpireUnreachable` (default: 30 days) protects unreachable reflog entries. Until both windows expire, any orphaned object is recoverable with `git cat-file -p <hash>`. Use `git gc --prune=now` to remove all unreachable objects immediately, bypassing the expiry windows.

**3.**
```bash
printf "blob 6\0hello\n" | shasum
```
Git hashes the full object string — type SP length NUL content — not the raw content alone. "hello\n" is 6 bytes (5 characters + 1 newline). The length in the header must match the exact byte count.

</details>

---

## Part 1 — How Git Represents Nested Directories

In Demo 02 the tree had two blobs — a flat structure.
Real projects look like this:

```
first-project/
├── file1.txt
├── file2.txt
└── src/
    ├── app.js
    └── utils/
        └── helpers.js
```

Git represents this as a hierarchy of tree objects:

```
┌─────────────────────────────────────────────────────────────────┐
│  ROOT TREE (represents first-project/)                          │
│                                                                  │
│  100644 blob xxxxxxxx    file1.txt                              │
│  100644 blob yyyyyyyy    file2.txt                              │
│  040000 tree zzzzzzzz    src          ← points to another TREE │
│                                                                  │
│       ┌──────────────────────────────────────────┐              │
│       │  TREE (represents src/)                  │              │
│       │                                          │              │
│       │  100644 blob aaaaaaaaa    app.js         │              │
│       │  040000 tree bbbbbbbbb    utils          │              │
│       │                                          │              │
│       │       ┌──────────────────────────┐       │              │
│       │       │  TREE (represents utils/)│       │              │
│       │       │                          │       │              │
│       │       │  100644 blob ccccccccc   │       │              │
│       │       │           helpers.js     │       │              │
│       │       └──────────────────────────┘       │              │
│       └──────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────┘

Key rules:
  • Every directory = one tree object
  • A tree entry with mode 040000 points to another tree (a subdirectory)
  • A tree entry with mode 100644 points to a blob (a file)
  • The root tree is what a commit object points to
  • Git never stores the directory path — only the name within the parent
```

### Why each directory is a separate object

```
If src/app.js changes:
  → New blob created for app.js (new content → new hash)
  → New tree created for src/   (now points to new app.js blob)
  → New tree created for root   (now points to new src/ tree)
  → New commit created          (points to new root tree)

file1.txt, file2.txt, and utils/ are UNCHANGED:
  → Their blobs are REUSED — same content, same hash
  → utils/ tree is REUSED — same content, same hash

This is Git's structural sharing. Only changed objects get new hashes.
Everything unchanged is reused by reference.
```

---

## Part 2 — Walkthrough: Building a Nested Tree Structure

We extend `first-project/` with a `src/` subdirectory.

### Step 1 — Navigate to first-project

```bash
cd ~/git-mastery-labs/04-trees-staging-area/src/first-project
```

### Step 2 — Create blobs for the new files

```bash
# Create blobs for src/ directory files
APP_BLOB=$(echo "console.log('hello from app');" | git hash-object --stdin -w)
HELPER_BLOB=$(echo "function helper() { return true; }" | git hash-object --stdin -w)

echo "app.js blob:      $APP_BLOB"
echo "helpers.js blob:  $HELPER_BLOB"

git cat-file --batch-all-objects --batch-check
# Shows all objects including the new blobs
```

### Step 3 — Create the utils/ tree

```bash
# Build the utils/ tree — contains helpers.js only
UTILS_TREE=$(printf "100644 blob $HELPER_BLOB\thelpers.js\n" | git mktree)
echo "utils/ tree: $UTILS_TREE"

# Verify
git cat-file -p $UTILS_TREE
# 100644 blob cccccc...    helpers.js

git cat-file -t $UTILS_TREE
# tree
```

### Step 4 — Create the src/ tree

```bash
# Build the src/ tree — contains app.js and points to utils/ tree
# Note: subdirectory entries use mode 040000 and type "tree"
SRC_TREE=$(printf "100644 blob $APP_BLOB\tapp.js\n040000 tree $UTILS_TREE\tutils\n" \
  | git mktree)
echo "src/ tree: $SRC_TREE"

# Verify
git cat-file -p $SRC_TREE
# 100644 blob aaaaaa...    app.js
# 040000 tree cccccc...    utils
```

### Step 5 — Create the new root tree

```bash
# Get the existing blob hashes from Demo 02
BLOB1=$(echo "Hello, git" | git hash-object --stdin)
BLOB2=$(git hash-object /tmp/new-file.txt 2>/dev/null || \
        echo "Second file in git repository" | git hash-object --stdin)

# Build the new root tree — includes original files + src/ subdirectory
ROOT_TREE=$(printf \
  "100644 blob $BLOB1\tfile1.txt\n100644 blob $BLOB2\tfile2.txt\n040000 tree $SRC_TREE\tsrc\n" \
  | git mktree)
echo "root tree: $ROOT_TREE"

# Verify the full structure
git cat-file -p $ROOT_TREE
# 100644 blob 8ab686...    file1.txt
# 100644 blob 44xxxx...    file2.txt
# 040000 tree xxxxxx...    src
```

### Step 6 — Walk the full tree hierarchy

```bash
# Root
git cat-file -p $ROOT_TREE

# src/ subtree (use the hash from the root tree's src entry)
SRC_HASH=$(git cat-file -p $ROOT_TREE | grep "src" | awk '{print $3}')
git cat-file -p $SRC_HASH

# utils/ subtree
UTILS_HASH=$(git cat-file -p $SRC_HASH | grep "utils" | awk '{print $3}')
git cat-file -p $UTILS_HASH

# Count all objects now
git cat-file --batch-all-objects --batch-check | wc -l
find .git/objects -type f | wc -l
```

---

## Part 3 — The Index in Depth

The index (`.git/index`) is a binary file that records the current
state of the staging area. It is the bridge between the working
directory and the repository.

### What the index stores per file

```
┌─────────────────────────────────────────────────────────────────┐
│  INDEX ENTRY — one per staged file                               │
│                                                                  │
│  ctime      last change time of the file on disk                │
│  mtime      last modification time of the file on disk          │
│  dev        device number (filesystem)                          │
│  ino        inode number                                        │
│  mode       file permission mode (e.g. 100644)                  │
│  uid        user ID of file owner                               │
│  gid        group ID of file owner                              │
│  size       file size in bytes                                  │
│  sha1       40-char hash of the blob object                     │
│  flags      stage number + name length                          │
│  name       file path relative to repo root                     │
│                                                                  │
│  WHY store ctime/mtime/inode?                                   │
│  git add hashes the file to create/update the blob.             │
│  On subsequent git status calls, Git checks ctime/mtime first.  │
│  If unchanged → skip rehashing → status is fast even for       │
│  large repos. Only changed files are rehashed.                  │
└─────────────────────────────────────────────────────────────────┘
```

### Reading the index

The index is binary — not readable with `cat`. Use these commands:

```bash
# List staged files (names only)
git ls-files

# List with permissions, hash, stage number, name
git ls-files -s

# List with status flags
git ls-files -v
# H = cached (normal)
# S = skip-worktree
# M = merge conflict (unmerged)

# Show files that would be ignored by .gitignore
git ls-files --others --ignored --exclude-standard

# Show untracked files (not yet git-added)
git ls-files --others
```

---

## Part 4 — Walkthrough: git add in Depth

Now we stop using plumbing commands and use `git add` — the porcelain
command. We observe exactly what it does to `.git/index`.

### Command introduction — git add

```
NAME
  git add — add file contents to the staging area (index)

SYNTAX
  git add <file>           stage a specific file
  git add <directory>      stage all files in a directory recursively
  git add .                stage all changes in the current directory
  git add -A               stage all changes including deletions everywhere
  git add -p               interactive — stage individual hunks (Demo 06)

WHAT IT DOES (internally)
  For each file:
  1. Reads the file from the working directory
  2. Computes SHA-1 hash of (blob header + file contents)
  3. If hash differs from index entry → writes new blob to .git/objects/
  4. Updates .git/index entry: new hash, new mtime, new size

  If the file content is unchanged since last add:
  → mtime check passes → blob already exists → index not updated
  → This is why git add is fast on large repos with few changes

OLD vs NEW commands:
  git add <file>         same in all versions — no change needed
  git rm --cached <file> (old) same as git restore --staged <file> (new, Git 2.23+)
```

### Step 7 — Start fresh with a clean working area

```bash
# Verify current state
git status
git ls-files -s
```

### Step 8 — Create real files in the working directory

```bash
# Create the directory structure
mkdir -p src/utils

# Create the files
echo "console.log('hello from app');" > src/app.js
echo "function helper() { return true; }" > src/utils/helpers.js
echo "# first-project" > README.md

# Check working directory vs staging area
git status
# Shows three untracked files — not yet in the index

git ls-files --others
# Lists untracked files: README.md, src/app.js, src/utils/helpers.js
```

### Step 9 — Stage files one at a time and observe the index

```bash
# Stage README.md only
git add README.md

# Inspect index after first add
git ls-files -s
# 100644 <hash>  0    README.md
# Only one entry — src/ files not yet staged

# Check object database — a new blob was created
git cat-file --batch-all-objects --batch-check | grep blob
# New blob for README.md content appears

git status
# Changes to be committed:  README.md
# Untracked files:          src/app.js, src/utils/helpers.js
```

### Step 10 — Stage the src/ directory

```bash
git add src/

# Inspect index now
git ls-files -s
# 100644 <hash>  0    README.md
# 100644 <hash>  0    src/app.js
# 100644 <hash>  0    src/utils/helpers.js
# Three entries — note: no entry for src/ or src/utils/
# The index tracks FILES only, not directories

git cat-file --batch-all-objects --batch-check
# New blobs for src/app.js and src/utils/helpers.js

git status
# Changes to be committed: README.md, src/app.js, src/utils/helpers.js
```

> **Important:** The index stores file paths, not directory structure.
> Directories themselves are not tracked — only the files inside them.
> The tree objects that represent directories are created at `git commit` time,
> not at `git add` time.

### Step 11 — Modify a staged file and observe the difference

```bash
# Modify a file that is already staged
echo "// added a comment" >> src/app.js

# Now check status
git status
# Changes to be committed:   src/app.js  ← the staged (old) version
# Changes not staged:        src/app.js  ← the working dir (new) version

# The same file appears in BOTH sections because:
# Index has: old content (before the appended line)
# Working dir has: new content (after the appended line)

git ls-files -s
# Index still shows the old hash for src/app.js

# To see exactly what is staged vs what is in working dir:
git diff --cached   # index vs last commit (what WILL be committed)
git diff            # working dir vs index (what will NOT be committed yet)
```

### Step 12 — Unstage a file

```bash
# OLD command (Git < 2.23):
git rm --cached src/app.js

# MODERN command (Git 2.23+):
git restore --staged src/app.js

# Both do the same thing:
# Removes the file from .git/index
# Does NOT delete the file from the working directory
# Does NOT delete the blob from .git/objects/

# Verify
git ls-files -s
# src/app.js is gone from the index

git status
# src/app.js is now "untracked" again

ls src/
# src/app.js still exists on disk — unstage ≠ delete
```

---

## Part 5 — git diff: Working Directory vs Staging Area

### Command introduction — git diff

```
NAME
  git diff — show changes between areas

SYNTAX AND WHAT IT COMPARES
  git diff                  working directory vs staging area (index)
                            shows: what you have changed but NOT yet staged

  git diff --cached         staging area (index) vs last commit
  git diff --staged         (same as --cached — both work)
                            shows: what WILL be committed next

  git diff <commit>         working directory vs a specific commit
  git diff <commit> <commit> changes between two commits

  git diff --stat           summary: files changed, insertions, deletions
  git diff --name-only      only filenames, no diff content

KEY INSIGHT — the three combinations:
  git diff            = "what have I changed that I haven't staged yet?"
  git diff --cached   = "what will my next commit contain?"
  git diff HEAD       = "what has changed since my last commit (staged + unstaged)?"
```

### Step 13 — Practice reading diff output

```bash
# Re-stage app.js (from Step 12 we unstaged it)
git add src/

# Modify a file after staging
echo "// another change" >> README.md

# Now demonstrate all three diff modes:

# What is staged (will be committed):
git diff --cached
# Shows README.md (original content staged) and src/ files

# What is NOT yet staged (in working dir but not index):
git diff
# Shows the "// another change" line added to README.md after staging

# Everything changed since last commit (staged + unstaged combined):
git diff HEAD
# Shows all changes
```

---

## Part 6 — The Index and Tree Creation at Commit Time

This part connects everything — showing exactly how the index becomes
a tree object when `git commit` runs. The actual commit is made in Demo 05.
Here we create the tree from the index using plumbing.

### Command introduction — git write-tree

```
NAME
  git write-tree — create a tree object from the current index

SYNTAX
  git write-tree

WHAT IT DOES
  Reads the current staging area (.git/index)
  Creates tree objects for every directory represented
  Returns the SHA-1 of the root tree

  This is exactly what git commit calls internally before
  creating the commit object. Understanding write-tree
  completes the picture of what git commit does.
```

### Step 14 — Write a tree from the current index

```bash
# Make sure everything is staged cleanly
git add .
git ls-files -s
# All files staged

# Create tree objects from the index
ROOT_TREE=$(git write-tree)
echo "Root tree hash: $ROOT_TREE"

# Inspect the tree hierarchy
git cat-file -p $ROOT_TREE
# 100644 blob xxxx    README.md
# 100644 blob xxxx    file1.txt
# 100644 blob xxxx    file2.txt
# 040000 tree xxxx    src

# The src/ subtree
SRC_SUBTREE=$(git cat-file -p $ROOT_TREE | grep " src$" | awk '{print $3}')
git cat-file -p $SRC_SUBTREE
# 100644 blob xxxx    app.js
# 040000 tree xxxx    utils

# utils/ subtree
UTILS_SUBTREE=$(git cat-file -p $SRC_SUBTREE | grep " utils$" | awk '{print $3}')
git cat-file -p $UTILS_SUBTREE
# 100644 blob xxxx    helpers.js
```

### Command introduction — git commit-tree

```
NAME
  git commit-tree — create a commit object from a tree hash

SYNTAX
  git commit-tree <tree-hash> -m "<message>"
  git commit-tree <tree-hash> -p <parent-hash> -m "<message>"

WHAT IT DOES
  Creates a commit object pointing to the given tree
  -p specifies the parent commit (omit for the first/root commit)
  Returns the SHA-1 of the new commit object
  Does NOT update HEAD or any branch ref — that is done separately

  This is the plumbing equivalent of git commit.
  In Demo 05 we use git commit-tree to build commits manually,
  then update HEAD to make them reachable from the branch.
```

> We create the actual commit in Demo 05. Here we stop at
> `git write-tree` to keep the focus on the tree and index.

---

## CLI Verification Toolkit

```bash
# ─────────────────────────────────────────────────────
# INSPECT THE INDEX
# ─────────────────────────────────────────────────────
git ls-files                      # tracked file names
git ls-files -s                   # detailed: mode | hash | stage | path
git ls-files -v                   # with status flags (H=normal, S=skip-worktree)
git ls-files --others             # untracked files
git ls-files --others --ignored --exclude-standard  # ignored files

# ─────────────────────────────────────────────────────
# COMPARE AREAS
# ─────────────────────────────────────────────────────
git diff                   # working dir vs index (unstaged changes)
git diff --cached          # index vs last commit (staged changes)
git diff --staged          # same as --cached
git diff HEAD              # working dir + index vs last commit (all changes)
git diff --stat            # summary view
git diff --name-only       # filenames only

# ─────────────────────────────────────────────────────
# ADD AND UNSTAGE
# ─────────────────────────────────────────────────────
git add <file>             # stage a file
git add .                  # stage all changes in current dir
git add -A                 # stage all changes including deletions
git restore --staged <file>  # unstage (modern, Git 2.23+)
git rm --cached <file>       # unstage (classic — still works)

# ─────────────────────────────────────────────────────
# CREATE TREE FROM INDEX (plumbing)
# ─────────────────────────────────────────────────────
git write-tree             # create tree object from current index
```

---

## Key Takeaways

1. **Every directory is a separate tree object — only changed directories get new hashes.** If `src/app.js` changes, Git creates a new blob for `app.js`, a new tree for `src/`, and a new root tree. All unchanged files and directories are reused by reference. This structural sharing keeps repository size manageable even with long histories.

2. **The index tracks files, not directories.** `.git/index` stores flat paths like `src/utils/helpers.js` — not separate entries for `src/` and `src/utils/`. Directory tree objects are reconstructed from these paths at commit time by `git write-tree`. Knowing this explains why `git add src/` stages files, not a tree object.

3. **`git add` creates blobs immediately — tree objects are created at commit time.** Every `git add` writes a blob to `.git/objects/` and updates the index entry with the new hash, mtime, and size. Running `git commit` then calls `git write-tree` to build tree objects from the index, followed by `git commit-tree` to wrap the root tree in a commit object.

4. **A file can legitimately appear in both staged and unstaged sections simultaneously.** Staged = the version in the index from the last `git add`. Unstaged = changes made to disk after that `git add`. Use `git diff --cached` to see what will be committed and `git diff` to see what will not yet be committed.

5. **`git restore --staged` and `git rm --cached` both unstage without deleting from disk.** The blob remains in `.git/objects/` and the file remains in the working directory. `git restore --staged` is the modern command (Git 2.23+); `git rm --cached` is the classic equivalent. Neither touches the blob or the working directory file.

### Quick reference — commands used in this demo

| Command | What it does |
|---|---|
| `git add <file>` | Stage a file — creates blob, updates index |
| `git add .` | Stage all changes in current directory |
| `git add -A` | Stage all changes including deletions everywhere |
| `git restore --staged <file>` | Unstage a file (modern, Git 2.23+) |
| `git rm --cached <file>` | Unstage a file (classic) |
| `git ls-files` | List staged files |
| `git ls-files -s` | Staged files with permissions, hash, stage, path |
| `git ls-files --others` | List untracked files |
| `git diff` | Working directory vs index (unstaged changes) |
| `git diff --cached` | Index vs last commit (staged changes) |
| `git diff HEAD` | All changes since last commit |
| `git diff --stat` | Summary of changes |
| `git write-tree` | Create tree object from current index |
| `git mktree` | Create tree object from explicit stdin input |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| File shows in both staged and unstaged | File modified after `git add` | `git add <file>` again to re-stage the latest version |
| `git diff --cached` shows nothing | Nothing staged yet | Run `git add` first |
| `git diff` shows nothing | No changes since last `git add` | Expected if working dir matches index |
| `git ls-files` shows unexpected files | Old index entries from previous adds | `git rm --cached <file>` to clean up |
| `git write-tree` fails | Merge conflicts in index (stage number > 0) | Resolve conflicts first — `git ls-files -s` shows stage numbers |
| Subdirectory blob not updating | Only parent directory staged | Use `git add src/` or `git add -A` to include nested changes |

---

## Break-Fix Scenario

```bash
cd ~/git-mastery-labs/04-trees-staging-area/src/break-fix
git init

# Create some files
echo "content one" > alpha.txt
echo "content two" > beta.txt
mkdir -p config
echo "key=value" > config/settings.txt
```

Now run each broken command, diagnose the error, and fix it:

```bash
# Command 1 — stage everything
git ADD .

# Command 2 — unstage beta.txt without deleting it from disk
git rm beta.txt

# Command 3 — show what is staged and ready to commit
git diff

# Command 4 — create a tree from the index
git write-tree --prefix=config/
```

<details>
<summary>Reveal answers — attempt diagnosis first</summary>

**Error 1 — `git ADD .` (uppercase ADD)**
Git commands are lowercase. `ADD` is not a valid command.
Git returns: `git: 'ADD' is not a git command`
Fix: `git add .`

**Error 2 — `git rm beta.txt` (missing --cached)**
`git rm` without `--cached` deletes the file from both the index AND
the working directory. The goal was to unstage only — leaving the file on disk.
Fix: `git rm --cached beta.txt`
OR the modern equivalent: `git restore --staged beta.txt`

After running the wrong command, restore the file:
`echo "content two" > beta.txt && git add beta.txt`

**Error 3 — `git diff` shows unstaged changes, not staged**
`git diff` compares working directory against the index.
It shows what has changed since the last `git add` — NOT what is staged.
To see what is staged (ready to commit): `git diff --cached`
The question asked for "what is staged and ready to commit".

**Error 4 — `git write-tree --prefix=config/`**
`git write-tree` does not accept a `--prefix` option.
Git returns: `error: unknown option 'prefix'`
`git write-tree` always creates a tree from the entire index.
To write only a subtree, use: `git ls-tree` (read) or structure the
index correctly before calling `write-tree`.
Fix: `git write-tree` (no flags needed)

</details>

---

## Interview Prep

**Q1. A colleague says "`git add` builds the directory tree Git stores." What is wrong with this and what is accurate?**
`git add` does not create tree objects. It creates blob objects for each file's contents and updates `.git/index` with a flat list of file paths, hashes, permissions, and timestamps. Tree objects — the objects that represent directory hierarchy — are created at commit time by `git write-tree`, which is called internally by `git commit`. A developer who believes trees are created on `git add` will be confused by why `.git/objects/` does not show new tree entries after staging and why tree hashes change only on commit.

**Q2. You run `git status` on a 50,000-file repository and it completes in under a second. How is that possible?**
The index stores `mtime` and `ctime` for every tracked file alongside the blob hash. On `git status`, Git checks the file's current `mtime` and `ctime` against the values in the index. If they match, the file is assumed unchanged and Git skips rehashing entirely — no file content read required. Only files with changed timestamps are rehashed. This timestamp-based short-circuit is what makes `git status` fast even on very large repositories.

**Q3. `git status` shows a file in both "Changes to be committed" and "Changes not staged for commit." A junior engineer says this is a Git bug. What is actually happening?**
Both entries are correct and intentional. "Changes to be committed" shows the version in the index — the snapshot captured by the last `git add`. "Changes not staged" shows modifications made to the working directory file after that `git add`. Git tracks both independently. The next `git commit` will use the staged version. To include the newer changes, run `git add <file>` again to update the index entry, then commit.

**Q4. What is the difference between `git diff`, `git diff --cached`, and `git diff HEAD`? When would you use each?**
`git diff` compares the working directory against the index — it shows what you have changed but not yet staged. Use this to review edits before `git add`. `git diff --cached` (or `--staged`) compares the index against the last commit — it shows exactly what the next `git commit` will include. Use this to review your staged changes before committing. `git diff HEAD` compares the working directory against the last commit, combining both staged and unstaged changes into one view. Use this to see everything changed since the last commit regardless of staging state.

---

## What's Next

**Demo 05 — Commits & The Three-Area Model**

```
✓ Create commit objects manually using git commit-tree
✓ Understand the commit chain — parent pointers and root commits
✓ Use git commit (porcelain) and understand exactly what it does
✓ Read any commit with git cat-file -p
✓ Understand HEAD and branch refs
✓ Use git log to traverse the commit chain
✓ See author vs committer difference in a live commit
```

---

## References

| Topic | Resource |
|---|---|
| git-add documentation | https://git-scm.com/docs/git-add |
| git-ls-files documentation | https://git-scm.com/docs/git-ls-files |
| git-diff documentation | https://git-scm.com/docs/git-diff |
| git-restore documentation | https://git-scm.com/docs/git-restore |
| git-write-tree documentation | https://git-scm.com/docs/git-write-tree |
| git-commit-tree documentation | https://git-scm.com/docs/git-commit-tree |
| Pro Git — The Index | https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified |

---

## Appendix — Anki Cards

**04-trees-staging-area-anki.csv:**

```
#deck:git-mastery-labs::Section 1 - Foundations::04-trees-staging-area
#separator:Comma
#columns:Front,Back,Tags
"A project has src/utils/helpers.js. How many tree objects does Git create to represent this path?","Three: one for the root directory, one for src/, and one for src/utils/. Each directory level is a separate tree object. The helpers.js file is a blob pointed to by the utils/ tree.","demo04 internals trees nested"
"You modify src/app.js after running git add src/app.js. git status shows app.js in BOTH staged and unstaged sections. Why?","The index has the version from the last git add (old content). The working directory has the newer version. Git tracks both: staged = what will be committed, unstaged = changes made after staging. Run git add src/app.js again to update the staged version.","demo04 staging index diff"
"What is the difference between git diff and git diff --cached (in a repo that has at least one commit)?","git diff: compares working directory against the index — shows unstaged changes (what you have changed but not yet added). git diff --cached: compares the index against the last commit — shows staged changes (what will be committed next).","demo04 commands diff staged"
"You run git add . on a directory containing src/utils/helpers.js. How many tree objects does git add create?","None. git add creates blob objects for each file and updates .git/index entries. Tree objects are only created at commit time by git write-tree (called internally by git commit).","demo04 internals add trees index"
"What does git restore --staged <file> do? What is the old equivalent?","Removes the file from the staging area (index) without deleting it from the working directory or .git/objects/. Old equivalent: git rm --cached <file>. Both produce the same result.","demo04 commands unstage restore rm-cached"
"Why does git status run fast even on repos with thousands of files?","The index stores mtime and ctime for each tracked file. On git status, Git checks these timestamps first. If unchanged since last add, it skips rehashing — no disk read of file content needed. Only files with changed timestamps are rehashed.","demo04 internals index performance"
"What does git write-tree do and when is it called?","Creates tree objects from the current staging area (index) and returns the root tree SHA-1. Called internally by git commit before creating the commit object. Can be called manually as a plumbing command.","demo04 commands write-tree plumbing"
"The index stores file paths but not directory entries. What does this mean for src/utils/helpers.js?","The index has one entry with path 'src/utils/helpers.js' — not separate entries for src/ and src/utils/. Directories are implied by path separators. Tree objects are reconstructed from these paths at commit time by git write-tree.","demo04 internals index paths"
"You want to see all files that exist in the working directory but are not tracked by Git. What command?","git ls-files --others — lists untracked files. Add --exclude-standard to also respect .gitignore rules: git ls-files --others --exclude-standard","demo04 commands ls-files untracked"
"In git ls-files -s output, every file shows stage number 0. What does stage number 0 mean?","Stage number 0 is the normal state — the file is tracked and not in a merge conflict. Stage numbers 1, 2, and 3 only appear during merge conflicts and represent the base, ours, and theirs versions respectively. Merge conflicts are covered in Demo 14.","demo04 commands ls-files stage-number"
```

---

## Appendix — Quiz

**04-trees-staging-area-quiz.md:**

````
# Quiz — Demo 04: Trees, Staging Area & Working Directory

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 05.

---

**Q1.** A project has this structure: `src/utils/helpers.js`.
How many Git tree objects are needed to represent this path?

- A) 1 — one tree for the entire project
- B) 2 — one for src/ and one for src/utils/
- C) 3 — one for root/, one for src/, one for src/utils/
- D) 0 — tree objects are only created when files change

<details>
<summary>Answer</summary>

**C** — Each directory level requires its own tree object.
Root tree → points to src/ tree → points to utils/ tree → points to helpers.js blob.
Three tree objects, one blob.

Trap: D is wrong — tree objects are created for every directory at
commit time regardless of whether files changed. However unchanged
tree objects may be reused by hash if their contents are identical.

</details>

---

**Q2.** You run `git add README.md` then edit `README.md` again.
`git status` shows `README.md` in both "Changes to be committed"
and "Changes not staged". What is the accurate explanation?

- A) Git is confused — run git add again to fix it
- B) The index has the pre-edit version; the working dir has the post-edit version
- C) README.md is corrupted and needs to be re-created
- D) This only happens if README.md has merge conflicts

<details>
<summary>Answer</summary>

**B** — The staging area (index) holds the snapshot from the last
`git add`. The working directory has the newer version. Both are valid
states. Run `git add README.md` again to update the staged version
to the current working directory content.

</details>

---

**Q3.** What does `git diff --cached` show?

- A) Changes in the working directory that have not been staged
- B) Changes between the staging area and the last commit
- C) Changes between two commits
- D) Changes between the working directory and the last commit

<details>
<summary>Answer</summary>

**B** — `git diff --cached` (also `git diff --staged`) compares
the index against the last commit (HEAD). It shows exactly what
will be included in the next `git commit`.

Trap: A describes `git diff` (no flags).
D describes `git diff HEAD`.

</details>

---

**Q4.** You run `git rm --cached config/settings.txt`. What happens?

- A) The file is deleted from disk and removed from the index
- B) The file is removed from the index only — it remains on disk
- C) The file is moved to a temporary stash area
- D) The file's blob is deleted from .git/objects/

<details>
<summary>Answer</summary>

**B** — `git rm --cached` removes the file from the staging area
(index) only. The file stays on disk in the working directory and
its blob stays in `.git/objects/`. After this, the file becomes
"untracked" from Git's perspective.

Trap: D is wrong — blob objects are never deleted by git rm.
They persist until git gc removes them after the expiry period.

</details>

---

**Q5.** When does `git add` create tree objects?

- A) Immediately — one tree per directory staged
- B) Never — git add only creates blobs and updates the index
- C) Only when using git add -A
- D) When the staged directory has more than 10 files

<details>
<summary>Answer</summary>

**B** — `git add` creates blob objects for file contents and updates
`.git/index` entries. Tree objects are created at commit time by
`git write-tree` (called internally by `git commit`). The index stores
flat file paths — trees are reconstructed from those paths on commit.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 5/5 | Import Anki CSV and move to Demo 05 |
| 4/5 | Review the wrong answer, then proceed |
| 3/5 | Re-read the relevant README section, retry quiz |
| Below 3/5 | Re-read Demo 04 and redo the walkthrough before proceeding |
````