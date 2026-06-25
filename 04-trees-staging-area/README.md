# Demo 04 — Trees, Staging Area & Working Directory

## Overview

In Demo 02 you built a flat tree with two blobs using plumbing commands.
Real projects are not flat — they have nested directories at multiple levels.
A Git tree object can point to other tree objects, creating a hierarchy that
mirrors your entire project structure.

This demo has two independent parts. Part 1 builds that nested structure
manually with plumbing commands so you see exactly how Git represents
directories internally, then introduces the two commands that move content
between the index and tree objects. Part 2 switches to the staging area as
a developer actually uses it day to day — `git add`, `git diff`, and
unstaging — so you understand the commands you will type in every real Git
session with complete confidence.

**What this demo covers:**

- Nested tree objects — trees pointing to trees
- How Git represents a full directory hierarchy as linked objects
- Why each directory is its own object — structural sharing in practice
- `git read-tree` — loading a tree object into the index
- `git write-tree` — turning a populated index into tree objects
- `git commit-tree` — introduced by name only; full walkthrough in Demo 05
- The staging area — what it is, and the commands that move content through it
- `git ls-files` — inspecting what is currently staged
- `git add` — what it does to `.git/index` step by step
- `git diff` — all three comparison modes, with real output
- `git rm --cached` — removing a file from the index, when it fails, and why

---

## Prerequisites

| Requirement | Detail | Verify |
|---|---|---|
| Demo 03 complete | Git object model and SHA-1 understood | `git cat-file -t <any-hash>` works |
| Git version | 2.28.0 or newer | `git --version` |
| Terminal | macOS/Linux or WSL2 | — |

> Demo 04 uses its own isolated repositories — `src/tree-demo/` for Part 1
> and `src/demo-app/` for Part 2. Neither depends on any repo from a
> previous demo.

---

## Concepts Covered

| Concept | What you will understand after this demo |
|---|---|
| Nested trees | How Git represents subdirectories as tree objects pointing to trees |
| Root tree | The top-level tree that a commit will eventually point to |
| Structural sharing | Why only changed directories get new tree objects |
| `git read-tree` | How a tree object's contents get loaded into the index |
| `git write-tree` | How a populated index becomes tree objects |
| `git commit-tree` (intro) | What it is for — full demonstration deferred to Demo 05 |
| The staging area | What it is, and the full set of commands that move content through it |
| `git ls-files` | Inspecting exactly what is currently staged, and how |
| `git add` internals | Exactly what happens in `.git/index` on every add, including `-A` behaviour |
| `git diff` | Three modes — working dir vs index, index vs last commit, all changes |
| `git rm --cached` | Removing a file from the index — the one case it refuses and why |

---

## Directory Structure

```
04-trees-staging-area/
├── README.md                          # this file
├── 04-trees-staging-area-anki.csv     # Anki flashcard deck
├── 04-trees-staging-area-quiz.md      # standalone quiz
└── src/
    ├── tree-demo/                     # Part 1 — isolated repo
    ├── demo-app/                      # Part 2 — isolated repo
    └── break-fix/                     # isolated break-fix environment
```

```
# .gitignore entries for this demo
src/tree-demo/
src/demo-app/
src/break-fix/
```

> All three directories under `src/` have their own `git init`. None depend
> on each other or on any prior demo's repository.

---

## Spaced Recall

*Answer from memory before reading anything below. No peeking at Demo 03.*

1. What is the difference between a loose object and a pack file?
2. You run `git gc`. Are unreachable objects deleted immediately? Why?
3. You want to reproduce the SHA-1 hash Git produces for a file
   containing "hello\n". What exact command do you run?

<details>
<summary>Reveal answers — attempt from memory first</summary>

**1.** A loose object is an individual zlib-compressed file stored at `.git/objects/<first2>/<remaining38>` — one file per object, created immediately on `git hash-object -w` or `git add`. A pack file combines many objects into a single binary file using delta compression. Pack files are created by `git gc` or during push/clone and are significantly more storage-efficient for repos with many similar object versions.

**2.** No — not immediately by default. `git gc` removes unreachable loose objects only after both expiry windows pass: `gc.pruneExpire` (default: 2 weeks) is the minimum age before a loose unreachable object is pruned, and `gc.reflogExpireUnreachable` (default: 30 days) protects unreachable reflog entries. Use `git gc --prune=now` to remove unreachable objects immediately.

**3.**
```bash
printf "blob 6\0hello\n" | shasum
```
Git hashes the full object string — type SP length NUL content. "hello\n" is 6 bytes. The length must match the exact byte count of the content.

</details>

---

## Part 1 — Nested Trees & Structural Sharing

### How Git represents nested directories

In Demo 02 the tree had two blobs — a flat structure with no subdirectories.
Real projects look like this:

```
demo-app/
├── README.md
├── .env.example
└── src/
    ├── app.js
    └── utils/
        └── helpers.js
```

Git represents this as a hierarchy of tree objects:

```
┌─────────────────────────────────────────────────────────────────┐
│  ROOT TREE (represents demo-app/ root)                          │
│                                                                  │
│  100644 blob xxxxxxxx    .env.example                           │
│  100644 blob yyyyyyyy    README.md                              │
│  040000 tree zzzzzzzz    src         ← points to another TREE  │
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
  # mode 100644 → regular file, points to a blob object
  # mode 040000 → directory, points to another tree object
  # Each tree stores only the names of its direct children
  # Git never stores full paths in tree objects — only leaf names
  # The root tree is what a commit object will eventually point to (Demo 05)
```

### Why each directory is a separate object

This structure is what enables Git's structural sharing — only changed
objects get new hashes; everything else is reused.

---

### Step 1 — Initialise the demo repository

```bash
cd ~/git-mastery-labs/04-trees-staging-area/src
mkdir tree-demo
cd tree-demo
git init

# Confirm clean state — no objects yet
find .git/objects -type f
# (no output)
```

### Step 2 — Create blob objects for every file

```bash
README_BLOB=$(printf "# Demo App\nA sample Node.js application.\n" \
  | git hash-object --stdin -w)
echo "README.md blob:      $README_BLOB"

ENV_BLOB=$(printf "PORT=3000\nDB_HOST=localhost\nDB_NAME=demoapp\n" \
  | git hash-object --stdin -w)
echo ".env.example blob:   $ENV_BLOB"

APP_BLOB=$(printf "const express = require('express');\nconst app = express();\napp.listen(3000);\n" \
  | git hash-object --stdin -w)
echo "src/app.js blob:     $APP_BLOB"

HELPER_BLOB=$(printf "function formatDate(d) { return d.toISOString(); }\nmodule.exports = { formatDate };\n" \
  | git hash-object --stdin -w)
echo "src/utils/helpers.js blob: $HELPER_BLOB"

# Confirm all 4 blobs exist
git cat-file --batch-all-objects --batch-check
```

> **Note:** these blobs exist only as objects in `.git/objects/` right now.
> No real files have been written to disk yet, and `.git/index` is still
> empty — Part 1 works entirely at the object level until Step 6.

### Step 3 — Create the utils/ tree

```bash
# Format: "mode SP type SP hash TAB name"
# The tab character before the filename is mandatory — git mktree requires it.
UTILS_TREE=$(printf "100644 blob $HELPER_BLOB\thelpers.js\n" | git mktree)
echo "utils/ tree: $UTILS_TREE"

git cat-file -p $UTILS_TREE
# 100644 blob ccccc...    helpers.js

git cat-file -t $UTILS_TREE
# tree
```

### Step 4 — Create the src/ tree

```bash
# Subdirectory entries use mode 040000 and type "tree"
SRC_TREE=$(printf "100644 blob $APP_BLOB\tapp.js\n040000 tree $UTILS_TREE\tutils\n" \
  | git mktree)
echo "src/ tree: $SRC_TREE"

git cat-file -p $SRC_TREE
# 100644 blob aaaaa...    app.js
# 040000 tree bbbbb...    utils
```

### Step 5 — Create the root tree

```bash
ROOT_TREE=$(printf \
  "100644 blob $ENV_BLOB\t.env.example\n100644 blob $README_BLOB\tREADME.md\n040000 tree $SRC_TREE\tsrc\n" \
  | git mktree)
echo "Root tree: $ROOT_TREE"

git cat-file -p $ROOT_TREE
# 100644 blob eeee...    .env.example
# 100644 blob rrrr...    README.md
# 040000 tree ssss...    src
```

### Step 6 — Walk the complete tree hierarchy

```bash
echo "=== ROOT TREE ==="
git cat-file -p $ROOT_TREE

echo "=== SRC/ TREE ==="
# Tree output uses a tab before the filename — match on it precisely
# to avoid matching unrelated entries that happen to contain "src"
SRC_HASH=$(git cat-file -p $ROOT_TREE | grep -P '\tsrc$' | awk '{print $3}')
git cat-file -p $SRC_HASH

echo "=== UTILS/ TREE ==="
UTILS_HASH=$(git cat-file -p $SRC_HASH | grep -P '\tutils$' | awk '{print $3}')
git cat-file -p $UTILS_HASH

# Total objects: 4 blobs + 3 trees = 7
git cat-file --batch-all-objects --batch-check | wc -l
# 7
```

> **Note on the grep pattern:** `git cat-file -p` output separates the
> hash from the filename with a tab character, not a space — `grep -P
> '\tsrc$'` matches the tab precisely. A plain `grep " src$"` looks for a
> literal space and will never match, silently producing empty output.

Confirm there are still no real files on disk and nothing staged — Part 1
has only written objects so far:

```bash
git ls-files
# (no output — index is empty, untouched since git init)

ls -l
# total 0   (no real files exist in the working directory yet)
```

>**Note:**If `src/app.js` changes,Git creates a new blob for `app.js`, a new tree for `src/` (because its contents changed), and a new root tree (because `src/`'s hash changed) — but `README.md`, `.env.example`, and the entire `src/utils/` subtree are reused without creating a single new object, because nothing about them changed. The deeper mechanics of exactly what happens to existing commits and objects when this kind of change occurs — what stays, what becomes unreachable, and when — is covered with concrete scenarios in Demo 05, once commits and parent pointers are part of the picture.

---

## Part 2 — Moving Content Between the Index and Trees

Part 1 built tree objects directly, one entry at a time, with `mktree`.
That is not how you will build trees in practice — in real use, the index
is populated first (by `git add`, or as you'll see here, by loading an
existing tree into it), and tree objects are derived from whatever the
index currently holds. This part covers the two plumbing commands that
do exactly that, in each direction.

### Command introduction — git read-tree

```
NAME
  git read-tree — load a tree object's contents into the index

SYNTAX
  git read-tree <tree-hash>

WHY THIS IS COVERED HERE
  Everything in Part 1 was built by hand, one tree entry at a time, with
  mktree — the index was never touched. git read-tree is the command that
  goes the other way: given a tree object that already exists, it populates
  the index with that tree's full contents. It is shown here so you have
  seen both directions — building a tree from nothing, and loading an
  existing tree into the index — before Part 2 moves on to git write-tree,
  which is the direction you will actually use in everyday Git.

WHAT IT DOES
  Reads the given tree object and all trees it points to, recursively.
  Replaces the current contents of .git/index with flat file-path entries
  for every blob in that tree hierarchy.
  Does NOT touch the working directory — files are not written to disk.
  Does NOT require any prior index state — it can populate an empty index
  or replace an already-populated one.

SCENARIOS WHERE THIS IS ACTUALLY USED
  git read-tree is mostly internal machinery, not something a developer
  runs directly day to day. It underlies operations like git checkout and
  git merge, where Git needs to load a tree from history into the index
  as part of switching branches or merging. You are running it directly
  here purely to see, explicitly, how a tree's contents get into the
  index — porcelain commands that switch branches do this same operation
  for you automatically.
```

### Step 7 — Load the tree you already built into the index

```bash
# Confirm the index is still empty before this command runs —
# Part 1 never wrote anything to .git/index
git ls-files
# (no output)

# Load the root tree from Part 1 into the index
git read-tree $ROOT_TREE

# Confirm the index now has all four files
git ls-files -s
# 100644 <hash>  0    .env.example
# 100644 <hash>  0    README.md
# 100644 <hash>  0    src/app.js
# 100644 <hash>  0    src/utils/helpers.js
```

> **Why this step has no real purpose in this exact sequence:** the tree
> you just loaded with `read-tree` is the same tree you already built by
> hand with `mktree` in Part 1 — `read-tree` here is not creating anything
> new or doing useful work in this specific walkthrough. It exists purely
> to demonstrate the mechanism. In real use, you would never `mktree` a
> tree and then immediately `read-tree` it back — you would populate the
> index by some other means (`git add`, or Git checking out a branch) and
> then move forward from there. The population path you will actually use
> day to day is `git add`, covered fully later in this demo.

---

### Command introduction — git write-tree

```
NAME
  git write-tree — create tree objects from the current staging area (index)

SYNTAX
  git write-tree

WHY THIS IS COVERED HERE
  This is the command direction you will actually rely on. Every real
  commit you ever make goes through this exact step internally: git commit
  calls git write-tree to turn whatever is currently staged into tree
  objects, before wrapping the result in a commit. You are about to use it
  directly so that when you see git commit do this invisibly in Demo 05,
  you already know exactly what is happening underneath it.

PRECONDITION
  The index must already be populated before calling git write-tree — it
  has nothing to work from otherwise. In this walkthrough, the index was
  just populated by git read-tree in Step 7. In real day-to-day use, the
  index is populated by git add instead — Part 2 of this demo covers that
  path in full once this section ends.

WHAT IT DOES
  Reads the current .git/index
  Reconstructs the directory hierarchy from the flat file paths stored there
  Creates one tree object per directory level
  Returns the SHA-1 hash of the root tree object

  Does NOT modify the index. Does NOT update HEAD or any branch ref —
  attaching a tree to history that way is what a commit object and a
  branch ref update do, both covered in full in Demo 05. write-tree's job
  ends at producing the tree object; nothing after that point is its
  responsibility.

  Same index content always produces the same tree hash — calling it
  twice with no changes between calls returns the identical hash.

SCENARIOS WHERE THIS IS ACTUALLY USED
  Every git commit, every time, without exception. It is the very first
  internal step git commit performs. It is also used directly by tools
  that need to inspect or compare a staged tree without creating a real
  commit — for example, scripts that want to diff "what's staged" against
  a specific historical tree.
```

### Step 8 — Reach the same tree a second way, via the index

```bash
# Load the existing tree into the index using git read-tree —
# this populates .git/index from the tree we already built
# git read-tree $ROOT_TREE                                      <<<Already done in Step 7>>>

# Confirm the index now has all four files
git ls-files -s
# 100644 <hash>  0    .env.example
# 100644 <hash>  0    README.md
# 100644 <hash>  0    src/app.js
# 100644 <hash>  0    src/utils/helpers.js

# Now ask git write-tree to build trees from this index
WRITE_TREE_RESULT=$(git write-tree)
echo "write-tree result: $WRITE_TREE_RESULT"
echo "original root tree: $ROOT_TREE"

# Both lines show the identical hash — same content, same tree, same SHA-1,
# reached two different ways: building tree objects by hand with mktree,
# versus letting write-tree derive them from a populated index.
```

---

### Command introduction — git commit-tree (intro only)

```
NAME
  git commit-tree — create a commit object from a tree hash

SYNTAX
  git commit-tree <tree-hash> -m "<message>"
  git commit-tree <tree-hash> -p <parent-hash> -m "<message>"

WHY THIS IS COVERED HERE, BRIEFLY
  You now have a fully verified, nested root tree object, confirmed two
  different ways. The next command you would reach for, to turn that tree
  into a real point in history, is git commit-tree. It is introduced here
  by name only — what it requires and what it produces — without being
  demonstrated yet, so you know it exists before Demo 05 gives it full
  treatment.

PRECONDITION
  Same as git write-tree — there must already be a tree object to wrap.
  You have one: $ROOT_TREE / $WRITE_TREE_RESULT from this part.

WHAT IT DOES, AT A HIGH LEVEL
  Wraps a tree object in a commit object — adding author, committer,
  timestamp, an optional parent pointer, and a message.
  Returns the SHA-1 of the new commit object.

  One detail worth flagging now: creating the commit object this way does
  NOT make it part of your repository's visible history by itself. A
  separate step — updating a branch ref (e.g. refs/heads/main) to point
  at the new commit hash — is what actually makes it reachable as "the
  current state of main." Without that second step, the commit object
  would exist in .git/objects/ but nothing would point to it.

  Both of these — the full git commit-tree walkthrough and what updating
  a branch ref actually means and does — are Demo 05's entire focus.
  Nothing here is repeated there; this is genuinely new material waiting
  for you in the next demo.

SCENARIOS WHERE THIS IS ACTUALLY USED
  Like write-tree, this runs internally on every git commit you make.
  Direct use is mostly limited to scripting and automation — for example,
  a CI pipeline programmatically constructing commits without a working
  directory checkout at all.
```

> **What's verified at this point:** `$ROOT_TREE` is a complete, correct,
> nested tree object, confirmed two different ways. Demo 05 picks up
> exactly here.

---

## Part 3 — The Staging Area in Daily Use

### What the staging area is

The staging area — `.git/index` — is the area between your working
directory and the repository. It holds a snapshot of exactly what your
next commit will contain, independent of whatever else is happening in
your working directory at the same moment. Nothing reaches a commit
without passing through it first.

There are both plumbing and porcelain commands for working with the
staging area. The table below is a map of all of them — what each does,
at a glance — with most explained in depth later in this demo, and the
remainder cross-referenced to where they are or will be covered.

| Action | Plumbing command | Porcelain command | Covered |
|---|---|---|---|
| Add/update content in the index | `git update-index` | `git add` | `git add` — in depth, later in this part. `git update-index` is the plumbing equivalent; not demonstrated directly in this series. |
| Remove content from the index | — | `git rm --cached` | In depth, later in this part |
| Restore index to HEAD version | — | `git restore --staged` | Not covered in this demo — this repo has no commits yet when this command would be used. Covered in Demo 06. |
| List index contents | — | `git ls-files` | In depth, immediately below |
| Build tree objects from the index | `git write-tree` | (internal to `git commit`) | Covered in Part 2 |
| Load a tree's contents into the index | `git read-tree` | (internal to `git checkout`, `git merge`) | Covered in Part 2 |
| Compare index against working dir or commit | `git diff-index`, `git diff-files` | `git diff`, `git diff --cached` | `git diff` — in depth, later in this part. The plumbing forms are not demonstrated in this series. |

This part covers `git ls-files`, `git add`, `git diff`, and `git rm --cached` in
full. The plumbing-only commands in the table above (`update-index`,
`diff-index`, `diff-files`) are included for completeness but are not
walked through directly — the porcelain commands that wrap them are what
you will use in practice. `git restore --staged` is listed in the table for
completeness but is not demonstrated here because it requires a resolvable
HEAD, which this demo's repo does not yet have at the relevant point — it
is covered in Demo 06.

### The index stores flat file paths, not directories

One fact has to be stated plainly before going further: **the index
stores a flat list of file paths — it has no entries for directories at
all.** A path like `src/utils/helpers.js` is one single index entry.
There is no separate entry for `src/` or `src/utils/`. Directories are
implied by path separators in the filename, nothing more. Tree objects —
the things that actually represent directories — are built from these
flat paths only when `git write-tree` runs, as you already saw in Part 2.

### What the index stores per file, and how it makes status fast

*(This binary format detail is introduced here, in the section it belongs
to — it is not repeated or duplicated anywhere else in this demo.)*

```
┌─────────────────────────────────────────────────────────────────┐
│  INDEX ENTRY — one entry per tracked file                        │
│                                                                  │
│  ctime    last status-change time of the file on disk           │
│  mtime    last modification time of the file on disk            │
│  dev      device number (filesystem identifier)                 │
│  ino      inode number                                          │
│  mode     file permission mode (e.g. 100644)                    │
│  uid      user ID of file owner                                 │
│  gid      group ID of file owner                                │
│  size     file size in bytes                                    │
│  sha1     hash of the blob object for this file version         │
│  flags    stage number + name length                            │
│  name     file path relative to repo root (e.g. src/app.js)    │
│                                                                  │
│  WHAT git status COMPARES, AND WHEN                              │
│                                                                  │
│  Step 1 — timestamp check (cheap, no file content read):        │
│  Git compares the working directory file's CURRENT ctime/mtime  │
│  against the ctime/mtime stored in this index entry.            │
│    → If both match: the file is assumed unchanged. Git trusts   │
│      the sha1 already stored in the index. STOP HERE — no       │
│      file content is read, no hash is computed.                 │
│    → If either differs: proceed to step 2.                      │
│                                                                  │
│  Step 2 — content check (expensive, only runs when step 1       │
│  finds a timestamp mismatch):                                   │
│  Git reads the file content, computes its SHA-1, and compares   │
│  that hash against the sha1 already stored in this index entry. │
│    → If the hash matches: content is genuinely unchanged —      │
│      only the timestamp changed (e.g. file was touched but      │
│      not edited). Git updates the stored timestamp so the next  │
│      status call can skip straight to step 1 again.             │
│    → If the hash differs: content really did change — reported  │
│      as modified.                                                │
│                                                                  │
│  On a repo with 50,000 tracked files where only 3 changed,      │
│  step 1 runs 50,000 times (cheap) but step 2 — the expensive    │
│  read-and-hash — runs only for the handful of files whose       │
│  timestamps actually moved. That is the entire performance      │
│  mechanism behind a fast git status.                             │
└─────────────────────────────────────────────────────────────────┘
```

---

### Command introduction — git ls-files

```
NAME
  git ls-files — show information about files in the index and working tree

SYNTAX
  git ls-files                              tracked file names only
  git ls-files -s                           mode, hash, stage number, path
  git ls-files -v                           tracked files with status flags
  git ls-files --others                     untracked files
  git ls-files --others --exclude-standard  untracked files, respecting .gitignore

WHAT EACH OPTION SHOWS
  (no flags)   Just the file paths currently in the index. Useful as a
               quick "what is staged" check.
  -s           Full detail per entry: file mode, blob SHA-1, stage number
               (0 = normal; 1/2/3 only appear during a merge conflict —
               covered in a later demo), and the path. This is the option
               used throughout this demo to inspect index state directly.
  -v           Adds single-letter status flags per file — H (cached/normal),
               S (skip-worktree, used in CI to ignore local changes to a
               file), among others.
  --others     Files Git sees in the working directory that are NOT in the
               index at all — i.e. untracked files.
  --others --exclude-standard
               Same as above, but filtered through .gitignore rules, so
               intentionally ignored files (build artifacts, node_modules,
               etc.) don't clutter the output.
```

### Step 9 — Initialise the second repo and set identity

```bash
cd ~/git-mastery-labs/04-trees-staging-area/src
mkdir demo-app
cd demo-app
git init

# This repo is isolated from every other repo in this demo and from any
# prior demo — per this demo's Prerequisites, nothing here depends on
# global config already being set elsewhere. Setting identity locally
# (no --global flag) keeps this repo fully self-contained and matches
# that isolation guarantee explicitly, even though your global identity
# from Demo 01 would otherwise apply here too.
git config user.name "Demo User"
git config user.email "demo@lab.local"
```

### Step 10 — Create the working directory files

```bash
mkdir -p src/utils

cat > README.md << 'EOF'
# Demo App
A sample Node.js application for Git Mastery Labs.
EOF

cat > .env.example << 'EOF'
PORT=3000
DB_HOST=localhost
DB_NAME=demoapp
EOF

cat > src/app.js << 'EOF'
const express = require('express');
const app = express();
app.listen(3000, () => console.log('Server started'));
EOF

cat > src/utils/helpers.js << 'EOF'
function formatDate(d) { return d.toISOString(); }
module.exports = { formatDate };
EOF

# Nothing staged yet
git status
# Untracked files: .env.example, README.md, src/

git ls-files
# (no output — index is empty)
```

---

### Command introduction — git add

```
NAME
  git add — add file contents to the staging area (index)

SYNTAX
  git add <file>           stage a specific file
  git add <directory>      stage all files in a directory, recursively
  git add .                stage all changes in the current directory,
                            and everything below it, recursively.
                            Includes new files, modified files, and
                            deleted files — but only within the current
                            directory subtree. Files in sibling or parent
                            directories are not affected.
  git add -A               stage all changes across the entire repository,
                            regardless of the current working directory.
                            Equivalent to: git add .; git add --deleted
                            from the repo root. Use -A when you want to
                            capture every change everywhere — including
                            deletions and renames — in a single command.

  git add . vs git add -A
  Both stage new and modified files recursively.
  The difference is scope and deletion handling:
    git add .   — stages from the current directory downward only.
                  Deletions in the current directory subtree are staged.
                  Files outside the current directory are untouched.
    git add -A  — stages everything in the entire repo root, from anywhere.
                  All deletions everywhere are staged.
                  Renames are detected (old path deleted, new path added)
                  regardless of which directory you are in when you run it.

WHAT IT DOES (internally)
  For each file:

  CASE 1 — First time staging this file (no existing index entry):
  1. Reads the file content from the working directory
  2. Computes SHA-1 of (blob header + file content)
  3. Writes a new compressed blob to .git/objects/
  4. Creates a new index entry: hash + mtime + ctime + size + path

  CASE 2 — Re-staging a previously staged file:
  1. Reads the file content from the working directory
  2. Computes SHA-1 of (blob header + file content)
  3. Compares computed hash to the hash stored in the existing index entry
     → If same hash: file content unchanged → nothing written to .git/objects/
       index entry mtime/ctime updated if timestamps changed, hash stays same
     → If different hash: new blob written to .git/objects/
       index entry updated with new hash, new mtime, new ctime, new size
  4. The previous blob remains in .git/objects/ until git gc removes it —
     this is exactly the orphaned-blob mechanism covered with concrete
     scenarios in Demo 05. The step you are about to run demonstrates the
     mechanism live, here, before Demo 05 names it formally.

  Does NOT create tree objects — those are created later by git write-tree
  (Part 2), called internally by git commit.
```

### Step 11 — Stage a file, then watch the orphan happen live

```bash
# Stage README.md — first time, Case 1
git add README.md

git ls-files -s README.md
# 100644 <hash-v1>  0    README.md

FIRST_README_BLOB=$(git ls-files -s README.md | awk '{print $2}')
echo "First README blob: $FIRST_README_BLOB"

# Confirm it is readable
git cat-file -p $FIRST_README_BLOB

# Now edit README.md and re-stage — Case 2, different hash
echo "## Getting Started" >> README.md
git add README.md

git ls-files -s README.md
# 100644 <hash-v2>  0    README.md   ← different hash now

SECOND_README_BLOB=$(git ls-files -s README.md | awk '{print $2}')
echo "Second README blob: $SECOND_README_BLOB"

# The index now points to the second blob only. Prove the first blob
# still physically exists and is still readable — before any commit exists:
git cat-file -p $FIRST_README_BLOB
# Still prints the original content — the blob was never deleted

git cat-file -t $FIRST_README_BLOB
# blob — still a valid object

# Confirm the second blob too — this is the content the index now
# actually points to, the one that will be staged going forward:
git cat-file -p $SECOND_README_BLOB
# Prints: # Demo App
#         A sample Node.js application for Git Mastery Labs.
#         ## Getting Started

# Nothing in the index, a ref, or a commit points to $FIRST_README_BLOB
# anymore. It is unreachable starting from the moment the second git add
# ran — not from any future commit. It will remain readable like this
# until git gc removes it after gc.pruneExpire passes.
```

### Step 12 — Stage the remaining files

```bash
git add .env.example src/

git ls-files -s
# 100644 <hash>  0    .env.example
# 100644 <hash>  0    README.md
# 100644 <hash>  0    src/app.js
# 100644 <hash>  0    src/utils/helpers.js

# Still no tree objects in .git/objects/ — only blobs.
# Confirmed by the index-tracks-files fact stated above this section.
git cat-file --batch-all-objects --batch-check
```

### Step 13 — Modify a staged file and see both states at once

```bash
# README.md is currently staged (version with "## Getting Started")
# Edit it again without re-staging
echo "## Installation" >> README.md

git status
# Changes to be committed:
#   new file:   README.md        ← the staged version (with Getting Started)
#
# Changes not staged for commit:
#   modified:   README.md        ← the working dir version (with Installation too)

# The index holds the staged snapshot. The working directory has gone
# further since then. Git tracks both independently — neither is wrong.
```

---

### Command introduction — git diff

```
NAME
  git diff — show changes between two areas, or between the working
             tree/index and any commit or ref

SYNTAX AND WHAT EACH MODE COMPARES

  git diff                    working directory vs staging area (index)
                              "What have I changed that I haven't staged yet?"

  git diff --cached           staging area (index) vs last commit (HEAD)
  git diff --staged           (identical — both flags mean the same thing)
                              "What will my next commit contain?"
                              With no commits yet: compares index vs an
                              implied empty tree, so every staged file
                              shows as a new addition.

  git diff HEAD               working directory vs last commit
                              "Everything changed since my last commit,
                               staged or not."
                              Requires at least one commit to exist —
                              HEAD cannot be resolved before that.
                              HEAD here is just the current example of a
                              more general capability: git diff <commit>
                              or git diff <ref> compares the working
                              directory and index against ANY commit or
                              ref you name, not only HEAD specifically.

HOW TO READ DIFF OUTPUT
  --- a/filename    the "before" version
  +++ b/filename    the "after" version
  -line             line removed from before version
  +line             line added in after version
```

### Step 14 — Run all three modes against the current state

```bash
# Mode 1 — what is NOT staged (working dir vs index):
git diff
# diff --git a/README.md b/README.md
# --- a/README.md
# +++ b/README.md
# @@ -2,3 +2,4 @@ A sample Node.js application for Git Mastery Labs.
#  ## Getting Started
# +## Installation
#
# Interpretation: README.md has an edit not yet staged. If you commit
# right now, "## Installation" will NOT be included.

# Mode 2 — what WILL be in the next commit (index vs last commit):
git diff --cached
# diff --git a/.env.example b/.env.example
# new file mode 100644
# --- /dev/null
# +++ b/.env.example
# @@ -0,0 +1,3 @@
# +PORT=3000
# +DB_HOST=localhost
# +DB_NAME=demoapp
#
# diff --git a/README.md b/README.md
# ... shows README.md WITH "## Getting Started" but WITHOUT "## Installation"
#
# diff --git a/src/app.js b/src/app.js
# ... (new file, full content)
#
# diff --git a/src/utils/helpers.js b/src/utils/helpers.js
# ... (new file, full content)
#
# Interpretation: this is exactly what the next commit will record.

# Mode 3 — everything since the last commit (requires a commit to exist):
git diff HEAD
# fatal: ambiguous argument 'HEAD': unknown revision or path not in the working tree.
#
# Expected — there is no commit yet in this repo, so HEAD cannot be
# resolved to anything. git diff HEAD becomes usable starting in Demo 05,
# right after the first commit exists.
```

> **The three questions to ask before every real commit:**
> 1. "What have I changed that isn't staged?" → `git diff`
> 2. "What will this commit contain?" → `git diff --cached`
> 3. "What changed in total since my last commit?" → `git diff HEAD` (needs a prior commit)

---

### Command introduction — git rm --cached

```
NAME
  git rm --cached — remove a file's entry from the staging area (index)
                    without deleting the file from the working directory

SYNTAX
  git rm --cached <file>
  git rm -f --cached <file>     # force — bypasses the one refusal case below

WHY
  Every file you stage with git add has an entry in .git/index tracking it.
  git rm --cached removes that entry — the file stops being tracked by Git
  from that point forward. The file itself remains on disk exactly as it is.
  This is distinct from git restore --staged (which restores the index entry
  to HEAD's version of the file rather than removing it) — git rm --cached
  does not need HEAD to exist at all, and it does not restore anything;
  it removes.

PRECONDITION
  No HEAD required — works in repos with zero commits.
  # Requires: a file currently in the index (git ls-files shows it)
  # Fails with: "error: the following file has staged content different
  #   from both the file and the HEAD" when all three areas disagree —
  #   the index version differs from both the working directory AND HEAD
  #   (or HEAD doesn't exist and the working directory has since diverged).
  #   In that case, the staged version exists nowhere else — Git refuses
  #   to discard it without explicit confirmation.
  # Force with: git rm -f --cached <file> to bypass this one protection.

SCENARIOS
  Remove a file from tracking without deleting it from disk:
    git rm --cached secrets.env
    # → secrets.env removed from index, still on disk, now untracked
    # → Next commit will record its deletion from the repository
    # → Common use: removing a file you accidentally staged

  Untrack a file and add it to .gitignore:
    git rm --cached node_modules/some-file.js
    echo "node_modules/" >> .gitignore
    git add .gitignore
    # → file no longer tracked, .gitignore prevents re-tracking

  Force-remove when index has diverged from working directory:
    git rm -f --cached config.txt
    # → use when you knowingly want to discard the staged version

WHAT IT DOES
  Removes the file's entry from .git/index.
  Does NOT delete the file from the working directory.
  Does NOT remove the blob from .git/objects/ — blobs are never deleted
  directly; git gc removes them after gc.pruneExpire if nothing references them.
  Does NOT update HEAD or any branch ref — the deletion only takes effect
  in Git's history when the next commit is made.
```

### Step 15 — Demonstrate git rm --cached and its one refusal case

```bash
# Current state from Step 13:
# Index: README.md staged at "## Getting Started" version
# Working directory: README.md has "## Getting Started" + "## Installation"
# No commits exist yet in this repo.

# Confirm what is staged and what is untracked before running anything
cat README.md
# # Demo App
# A sample Node.js application for Git Mastery Labs.
# ## Getting Started
# ## Installation

git ls-files README.md
# README.md   ← still staged, nothing changed

git status
# Changes to be committed:
#   new file:   README.md   ← staged version (with ## Getting Started)
#
# Changes not staged for commit:
#   modified:   README.md   ← working directory version (with ## Installation too)
#
# Two versions of README.md exist simultaneously:
# the staged version in the index, and the modified version on disk.

# Attempt git rm --cached — this hits the refusal case:
# index(v2 with Getting Started) ≠ working dir(v3 with Installation) ≠ HEAD(doesn't exist)
# All three areas disagree — staged version exists nowhere else
git rm --cached README.md
# error: the following file has staged content different from both the
# file and the HEAD:
#     README.md
# (use -f to force removal)
#
# This is the one protection git rm --cached has: the staged version of
# README.md ("## Getting Started") exists ONLY in the index. It is not
# in HEAD (no commits yet) and it is not in the working directory (the
# disk has a further-edited version). Removing the index entry without
# -f would permanently discard that staged snapshot.

# Force past the protection
git rm -f --cached README.md
# rm 'README.md'

git status
# Untracked files:
#   README.md
#
# The staged version has been removed from the index.
# README.md is now only an untracked file on disk.

ls README.md
# README.md   ← still on disk, not deleted

cat README.md
# # Demo App
# A sample Node.js application for Git Mastery Labs.
# ## Getting Started
# ## Installation
#
# The working directory version (with both appended lines) is intact.
# The staged version (with "## Getting Started" only) existed only in
# the index and is now gone — the blob object remains in .git/objects/
# as an orphan until git gc removes it.
```

> **`git restore --staged` is not demonstrated here.** It restores the
> index entry to HEAD's version of the file. This repo has no commits yet
> — there is no HEAD. `git restore --staged` fails unconditionally in
> this environment with `fatal: could not resolve HEAD`. It is covered in
> Demo 06, where commits exist and HEAD can be resolved.

### Step 16 — Re-stage cleanly and verify the final state in full

```bash
git add -A
git ls-files -s
# All four files staged at their current working directory versions

# Note: nothing has been committed yet at this point — git status
# confirms this explicitly:
git status
# On branch main
# No commits yet
# Changes to be committed: ...

# Confirm the current object inventory — only blobs exist at this point.
# No tree objects have been created yet; git write-tree has not run.
git cat-file --batch-all-objects --batch-check
# <hash> blob <size>    ← .env.example
# <hash> blob <size>    ← README.md (current version)
# <hash> blob <size>    ← README.md (orphaned first blob from Step 11)
# <hash> blob <size>    ← src/app.js
# <hash> blob <size>    ← src/utils/helpers.js
# (and any other blobs from intermediate staging during this demo)
# No tree entries — trees don't exist yet.

# Build the tree from this index — same command, same mechanics as
# Part 2, just driven by a real day-to-day staging session instead of
# git read-tree.
FINAL_TREE=$(git write-tree)

# Now check the object inventory again — three tree objects have appeared
git cat-file --batch-all-objects --batch-check
# <hash> blob <size>    ← (all blobs as before — unchanged)
# <hash> tree 107       ← root tree (NEW — just created by write-tree)
# <hash> tree 66        ← src/ tree (NEW — just created by write-tree)
# <hash> tree 38        ← utils/ tree (NEW — just created by write-tree)
#
# write-tree created exactly 3 new objects: one tree per directory level.
# The blobs were all created earlier by git add — write-tree never creates blobs.

echo "=== Verify the complete tree structure ==="
git cat-file -p $FINAL_TREE
# 100644 blob <hash>    .env.example
# 100644 blob <hash>    README.md
# 040000 tree <hash>    src

SRC_HASH=$(git cat-file -p $FINAL_TREE | grep -P '\tsrc$' | awk '{print $3}')
git cat-file -p $SRC_HASH
# 100644 blob <hash>    app.js
# 040000 tree <hash>    utils

UTILS_HASH=$(git cat-file -p $SRC_HASH | grep -P '\tutils$' | awk '{print $3}')
git cat-file -p $UTILS_HASH
# 100644 blob <hash>    helpers.js

echo "=== Full object inventory after write-tree ==="
git cat-file --batch-all-objects --batch-check
```

Example full object inventory after `git write-tree`, with types labelled:

```
56bb1d1874e2e5d74f778d73507e1760919e284d blob 84
6441d0e7f45455047591709df43ebb6b7aed26d3 tree 107
801f807cbccde81844ea1342111a1c28ddd06512 blob 114
b1643a9e435940b4947245ccd31c95957a33822c blob 81
bff4f9011eb78b351a055eebe8ed3a6b9f8ab835 blob 97
d93c602da85daf4849380c3c88280212fb040989 blob 44
e311e5b5f4d260d790e402ec6cc0e2851a4991cd tree 66
f7f3b240e113fd87dd5e7ce903d15f284aa5fde8 blob 62
fa8b95b1305bd38a3b87cde9d6f88aba60121869 tree 38
```

> **What changed because of `write-tree`, specifically:** before this
> command ran, `.git/objects/` held only blobs — one per `git add`
> performed across this part of the demo, including the orphaned first
> README blob from Step 11. The three `tree` entries in this listing
> (107, 66, and 38 bytes) are new, and they exist only because
> `write-tree` just created them — one for `utils/`, one for `src/`, and
> one for the root.
>
> **Is anything hanging or unreferenced in this specific listing?** No —
> every object shown here is reachable: the three new trees form the
> hierarchy just built, and every blob is either currently referenced by
> the index directly, or referenced by one of those trees. The one
> deliberately orphaned object from this demo — the first README blob
> from Step 11 — was already orphaned the moment it was superseded in
> the index, well before this step; this listing does not distinguish
> "orphaned but not yet gc'd" objects from reachable ones; all loose
> objects appear here regardless of reachability until `git gc` removes
> the unreachable ones.
```

---

## CLI Verification Toolkit

```bash
# ─────────────────────────────────────────────────────
# BUILD AND INSPECT TREES (Part 1 & Part 2)
# ─────────────────────────────────────────────────────
git hash-object --stdin -w     # create a blob from stdin content
git mktree                     # create one tree object from explicit stdin entries
git read-tree <tree-hash>      # load a tree object's contents into the index
git write-tree                 # create tree objects from the current index
git cat-file -p <hash>         # print object contents (any type)
git cat-file -t <hash>         # print object type
git cat-file --batch-all-objects --batch-check   # list every object with type+size

# ─────────────────────────────────────────────────────
# INSPECT THE INDEX (Part 3)
# ─────────────────────────────────────────────────────
git ls-files                      # tracked file names
git ls-files -s                   # mode | hash | stage | path
git ls-files -v                   # with status flags
git ls-files --others             # untracked files
git ls-files --others --exclude-standard  # untracked, respecting .gitignore

# ─────────────────────────────────────────────────────
# COMPARE AREAS
# ─────────────────────────────────────────────────────
git diff                   # working dir vs index (not yet staged)
git diff --cached          # index vs last commit (will be committed)
git diff HEAD              # working dir + index vs last commit (needs a commit)
git diff <commit-or-ref>   # working dir + index vs any named commit or ref
git diff --stat            # summary: files + lines changed
git diff --name-only       # filenames only

# ─────────────────────────────────────────────────────
# STAGE AND UNSTAGE
# ─────────────────────────────────────────────────────
git add <file>                       # stage a file (recursive if a directory)
git add .                            # stage all changes from here down, recursively
git add -A                           # stage all changes including deletions, repo-wide
git rm --cached <file>               # remove from index; keeps file on disk; no HEAD needed
git rm -f --cached <file>            # force remove — bypasses the one refusal case
```

---

## Key Takeaways

1. **Nested directories in Git are trees pointing to trees — each directory level is a separate tree object.** If `src/app.js` changes, Git creates a new blob for `app.js`, a new tree for `src/`, and a new root tree. Everything else is reused by hash. This structural sharing keeps repositories compact even with long histories.

2. **`git read-tree` and `git write-tree` move content between the index and tree objects in opposite directions.** `read-tree` loads an existing tree's contents into the index. `write-tree` derives tree objects from whatever the index currently holds. Every real `git commit` calls `write-tree` internally — `read-tree` underlies operations like `git checkout` and `git merge` rather than being something you typically run directly.

3. **`git write-tree` requires a populated index — it has nothing to build from otherwise.** In day-to-day use, the index gets populated by `git add`. The result is identical regardless of how the index got populated: same index content always produces the same tree hash.

4. **The index stores flat file paths, not directories.** `.git/index` has an entry for `src/utils/helpers.js` — no separate entries for `src/` or `src/utils/`. Directory tree objects are reconstructed from these flat paths only when `git write-tree` runs.

5. **`git status` is fast because of a two-step check, not a single shortcut.** Git first compares the file's current timestamps against the index — if they match, it trusts the stored hash with no file read at all. Only on a timestamp mismatch does Git read the file and compute a real hash to confirm whether content actually changed.

6. **`git rm --cached` removes a file from the index without touching the working directory — it does not need HEAD to exist.** The one case where it refuses is when the index version differs from both the working directory and HEAD simultaneously: that staged version exists nowhere else, so Git requires `-f` to confirm you understand you are discarding it.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `grep " src$"` on tree output returns nothing | `cat-file -p` separates hash and filename with a tab, not a space | Use `grep -P '\tsrc$'` to match the tab precisely |
| File shows in both staged and unstaged | File modified after `git add` | `git add <file>` again to update the staged version |
| `git diff --cached` shows nothing | Nothing staged | Run `git add` first |
| `git diff HEAD` errors: "ambiguous argument HEAD" | No commits exist yet | Expected pre-Demo-05 — use `git diff` and `git diff --cached` instead |
| `git rm --cached` errors: "staged content different" | Index version differs from both working directory and HEAD (or no HEAD and working dir has diverged) — staged version exists nowhere else | Use `git rm -f --cached <file>` to force removal, confirming you accept that staged version will be discarded |
| `git write-tree` fails | Merge conflict entries in index (stage number > 0) | Resolve conflicts — `git ls-files -s` shows entries with stage 1/2/3 |
| Object still readable after re-staging over it | Expected — blobs are never deleted until `git gc` runs past `gc.pruneExpire` | Not an error — confirms the orphan mechanism shown in Part 3 |

---

## Break-Fix Scenario

```bash
cd ~/git-mastery-labs/04-trees-staging-area/src/break-fix
git init
git config user.name "Demo User"
git config user.email "demo@lab.local"

cat > server.js << 'EOF'
const http = require('http');
http.createServer((req, res) => res.end('ok')).listen(8080);
EOF

cat > config.json << 'EOF'
{ "port": 8080, "debug": false }
EOF
```

Now run each broken command, diagnose the error from the output alone, and fix it:

```bash
# Command 1 — build a tree entry for server.js
echo "100644 blob $(git hash-object -w server.js) server.js" | git mktree

# Command 2 — stage everything
git ADD -A

# Command 3 — see what will be in the next commit
git diff

# Command 4 — remove config.json from staging without deleting from disk
git rm config.json
```

<details>
<summary>Reveal answers — attempt diagnosis first</summary>

**Error 1 — missing tab separator in the `mktree` entry**

Git returns: `fatal: input format error: not enough fields, or ill-formed entry`

`git mktree` requires a tab character between the hash and the filename — not a space. The `echo` here produced `"...blob <hash> server.js"` with a space, which `mktree` cannot parse as a valid entry.

Fix: use `printf` with an explicit `\t`:
```bash
printf "100644 blob $(git hash-object -w server.js)\tserver.js\n" | git mktree
```

---

**Error 2 — `git ADD -A` (uppercase ADD)**

Git returns: `git: 'ADD' is not a git command. See 'git --help'.`

Git commands are case-sensitive and always lowercase.

Fix: `git add -A`

---

**Error 3 — `git diff` shows nothing when the question asked about the next commit**

`git diff` with no flags compares the working directory against the index — it shows unstaged changes. There are none right now since everything was just staged in Command 2. The question asked what the next commit will contain, which requires:

Fix: `git diff --cached`

---

**Error 4 — `git rm config.json` (missing `--cached` flag)**

Git returns:
```
error: the following file has local modifications:
    config.json
(use --cached to keep the file, or -f to force removal)
```

`git rm` without `--cached` attempts to delete the file from both the index
AND the working directory. Git refuses because `config.json` has staged
content and would be deleted from disk.

The goal was to remove it from the index while keeping it on disk.

Fix: `git rm --cached config.json`

Verify the file is still on disk afterward: `ls config.json`

</details>

---

## Interview Prep

**Q1. A colleague says two commits with the same file content but different commit messages must have different tree hashes too. Are they right?**
No. A tree object's hash depends only on its entries — file names, modes, and blob/tree hashes of its children. The commit message lives in the commit object, not the tree. Two commits can point to the identical tree hash if nothing about the file content or structure changed between them — only the commit objects themselves differ, because they carry different messages, timestamps, or parent pointers. This is exactly what Part 2 demonstrated: the same root tree reached via `mktree`+`read-tree` and via `write-tree` produced the identical hash both times, because tree identity is purely about content, never about how it was built or what gets said about it later.

**Q2. You run `git rm --cached config.json` and it fails with "staged content different from both the file and the HEAD." What is happening, and what are your options?**
The error fires when three conditions are simultaneously true: the index holds a version of `config.json`, the working directory has a different version (edited since the last `git add`), and HEAD either doesn't exist or has yet another different version. In that state, the staged snapshot exists nowhere else — removing the index entry without warning would permanently discard it. Two options: `git rm -f --cached config.json` to force the removal and accept that the staged version is gone (use this when you've confirmed the staged version is not needed), or `git add config.json` first to bring the index version in sync with the working directory, then `git rm --cached config.json` will succeed without force.

**Q3. You want to stop tracking a file that already exists in the repository without deleting it from disk. What is the exact sequence of commands?**
First, remove it from the index: `git rm --cached <filename>`. This removes the index entry immediately and keeps the file on disk. Second, add the filename to `.gitignore` to prevent it from being re-staged accidentally: `echo "<filename>" >> .gitignore`. Third, stage the `.gitignore` update and commit: `git add .gitignore && git commit -m "Stop tracking <filename>"`. At this point, the file is on disk, no longer tracked by Git, and protected from being re-added by `.gitignore`. The distinction from `git rm <filename>` (no `--cached`) is critical — without `--cached`, `git rm` deletes the file from disk.

**Q4. Why does `git status` stay fast on a repository with tens of thousands of tracked files, even though it appears to check every single one?**
Because the expensive part — reading file content and computing a SHA-1 — only happens for files whose timestamps changed. Git's first pass compares each file's current `ctime`/`mtime` against the values already stored in the index entry; if they match, Git trusts the stored hash without touching the file's content at all. Only on a timestamp mismatch does Git fall through to actually reading the file and rehashing it, to confirm whether the content genuinely changed or just got touched. On a repo where 3 files out of 50,000 actually changed, the cheap timestamp comparison runs 50,000 times but the expensive read-and-hash runs only 3 times.

---

## What's Next

**Demo 05 — Commits & The Three-Area Model**

```
✓ Take the verified root tree from this demo and wrap it in a real commit
✓ Use git commit-tree fully — root commits, parent pointers, the commit chain
✓ See exactly what happens to existing commits and objects when a tracked
  file changes, both before and after the first commit exists
✓ Use git commit (porcelain) and understand exactly what it does internally
✓ Understand HEAD and branch refs — what they are, where they live on disk
✓ Use git log to traverse the commit chain
✓ See author vs committer differ in a live, hands-on commit
```

---

## References

| Topic | Resource |
|---|---|
| git-add documentation | https://git-scm.com/docs/git-add |
| git-ls-files documentation | https://git-scm.com/docs/git-ls-files |
| git-diff documentation | https://git-scm.com/docs/git-diff |
| git-rm documentation | https://git-scm.com/docs/git-rm |
| git-write-tree documentation | https://git-scm.com/docs/git-write-tree |
| git-read-tree documentation | https://git-scm.com/docs/git-read-tree |
| git-mktree documentation | https://git-scm.com/docs/git-mktree |
| Git index format specification | https://git-scm.com/docs/index-format |
| Pro Git — The Index | https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified |

---

## Appendix — Anki Cards

**04-trees-staging-area-anki.csv:**

```
#deck:git-mastery-labs::Section 1 - Git Foundations & Internals::04-trees-staging-area
#separator:Comma
#columns:Front,Back,Tags
"A project has src/utils/helpers.js. How many tree objects does Git need to represent this path?","Three: one for the root directory, one for src/, and one for src/utils/. Each directory level is a separate tree object. The helpers.js file itself is a blob pointed to by the utils/ tree.","demo04,internals,trees,nested"
"What is the difference between git read-tree and git write-tree?","They move content in opposite directions. git read-tree loads an existing tree object's contents into the index. git write-tree derives tree objects from whatever is currently in the index. write-tree is what every git commit calls internally; read-tree underlies operations like git checkout and git merge rather than being run directly day to day.","demo04,commands,read-tree,write-tree"
"What must be true of the index before calling git write-tree?","The index must already be populated — git write-tree has nothing to build a tree from otherwise. In real use, the index is populated by git add. Same index content always produces the same tree hash, regardless of how that index was populated.","demo04,commands,write-tree,precondition"
"Does git commit-tree update HEAD or any branch ref when it creates a commit object?","No. git commit-tree only creates the commit object itself in .git/objects/. Making that commit reachable as 'the current state of a branch' requires a separate step — updating a branch ref (e.g. refs/heads/main) to point at the new commit hash. Without that step, the commit object exists but nothing points to it. Covered in full in Demo 05.","demo04,commands,commit-tree,head,forward-reference"
"What is the difference between git diff and git diff --cached?","git diff compares the working directory against the index — shows changes not yet staged (will NOT be in the next commit unless added). git diff --cached compares the index against the last commit — shows exactly what the next commit will contain. git diff HEAD combines both and requires at least one commit to exist. HEAD is just one example of a more general git diff <commit-or-ref> capability.","demo04,commands,diff,staged"
"You run git add . on a directory containing src/utils/helpers.js. How many tree objects does git add create?","None. git add creates blob objects for file contents and updates .git/index entries with flat file paths. Tree objects (src/ tree, utils/ tree, root tree) are created later by git write-tree, called internally by git commit.","demo04,internals,add,trees,index"
"What is the difference between git add . and git add -A?","Both stage new and modified files recursively. The differences: git add . stages only from the current directory downward — files outside the current directory subtree are untouched. git add -A stages all changes across the entire repository regardless of which directory you are in, including all deletions and renames everywhere. Use git add -A from the repo root when you want to capture every change in one command.","demo04,commands,add,add-A"
"In a repository with no commits yet, you run git rm --cached config.json. It succeeds. Why does it not need HEAD?","git rm --cached removes the index entry for a file directly — it does not need HEAD to exist because it is not restoring anything. It simply removes the entry from .git/index. It only refuses (with an error) when all three areas disagree simultaneously: index, working directory, and HEAD (or the absence of HEAD when the working directory has diverged). When the working directory still matches what is staged, git rm --cached succeeds with zero commits in the repo.","demo04,commands,rm-cached,head,no-commits"
"git rm --cached config.json fails with 'staged content different from both the file and the HEAD'. What exactly is happening and what are the two options?","The index holds a version of config.json, the working directory has a different (more edited) version, and HEAD either doesn't exist or has yet another version. The staged snapshot exists nowhere else — removing it would permanently discard it. Option 1: git rm -f --cached config.json to force removal (use when you confirm the staged version is not needed). Option 2: git add config.json first to sync the index with the working directory, then git rm --cached config.json succeeds without force.","demo04,commands,rm-cached,error,force"
"What is the difference between git rm config.json and git rm --cached config.json?","git rm config.json deletes the file from both the index AND the working directory — the file is removed from disk. git rm --cached config.json removes the file's entry from the index only — the file stays on disk unchanged. Without --cached, Git also requires -f if the file has local modifications. Use --cached when you want to stop tracking a file without deleting it.","demo04,commands,rm,rm-cached,difference"
"Why does git status run fast on large repositories with tens of thousands of files?","git status first compares each file's current ctime/mtime against the values stored in the index. If they match, the file is assumed unchanged and Git trusts the stored hash with no file read at all. Only on a timestamp mismatch does Git read the file and compute a real hash to confirm whether content changed. The expensive read-and-hash step runs only for files whose timestamps moved, not for every tracked file.","demo04,internals,index,performance,mtime"
"The index stores flat file paths. What does this mean for src/utils/helpers.js?","The index has one entry with path src/utils/helpers.js — not separate entries for the directories src/ and src/utils/. Directory names appear only as prefixes of file paths. The tree objects that represent src/ and src/utils/ as directories are created from these path prefixes only when git write-tree runs.","demo04,internals,index,paths"
"What does git ls-files -s show, compared to plain git ls-files?","Plain git ls-files shows only tracked file paths. git ls-files -s adds, per file: the file mode, the blob SHA-1 hash, and the stage number (0 for a normal tracked file; 1/2/3 only appear during an unresolved merge conflict). -s is the option used to verify exactly what the index currently points to for each file.","demo04,commands,ls-files"
"git diff HEAD fails with 'fatal: ambiguous argument HEAD' — what is the cause?","The repository has no commits yet. git diff HEAD tries to resolve HEAD to a commit hash, but HEAD cannot resolve until the first commit exists. git diff and git diff --cached both work without any commits — git diff --cached compares the index against an implied empty tree. git diff HEAD becomes usable after the first commit, in Demo 05.","demo04,commands,diff,head,no-commits"
"What does git commit-tree create, and is it demonstrated fully in this demo?","git commit-tree wraps a tree object in a commit object — adding author, committer, an optional parent pointer, and a message. It is introduced by name and syntax only in this demo. The full demonstration — root commits, parent pointers, and the commit chain — is covered in detail in Demo 05.","demo04,commands,commit-tree,forward-reference"
```

---

## Appendix — Quiz

**04-trees-staging-area-quiz.md:**

````markdown
# Quiz — Demo 04: Trees, Staging Area & Working Directory

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 05.

---

**Q1.** A Node.js project has `src/utils/helpers.js`.
How many Git tree objects are needed to represent this path in a commit?

- A) 1 — one tree for the entire project
- B) 2 — one for `src/` and one for `src/utils/`
- C) 3 — one for root, one for `src/`, one for `src/utils/`
- D) 0 — tree objects are created by `git add`, which hasn't run yet

<details>
<summary>Answer</summary>

**C** — Each directory level requires its own tree object: root tree →
entry pointing to `src/` tree → entry pointing to `utils/` tree → entry
pointing to `helpers.js` blob. Three tree objects, one blob.

Trap: D confuses two separate facts — tree objects are not created by
`git add` (they come later, from `git write-tree`), but the question asks
how many are *needed* to represent the path, not when they get created.

</details>

---

**Q2.** What is the relationship between `git read-tree` and `git write-tree`?

- A) They do the same thing — either can be used interchangeably
- B) `read-tree` loads a tree's contents into the index; `write-tree` derives tree objects from the index
- C) `read-tree` is the modern replacement for `write-tree`
- D) `write-tree` requires `read-tree` to have run first, always

<details>
<summary>Answer</summary>

**B** — They move content in opposite directions between the index and
tree objects. `read-tree` populates the index from an existing tree.
`write-tree` builds tree objects from whatever the index currently holds,
regardless of how it got populated — by `read-tree`, by `git add`, or any
other means.

Trap: D is wrong — `write-tree` only requires *some* populated index,
not specifically one populated by `read-tree`. In real use, `git add` is
what populates it.

</details>

---

**Q3.** You run `git commit-tree <tree-hash> -m "message"` successfully.
Does this make the new commit part of your repository's visible history?

- A) Yes — it automatically becomes the new HEAD
- B) Yes — it updates the current branch automatically
- C) No — a separate step is required to point a branch ref at the new commit
- D) No — `git commit-tree` cannot create valid commits without a working directory

<details>
<summary>Answer</summary>

**C** — `git commit-tree` only creates the commit object in
`.git/objects/`. Nothing about HEAD or any branch changes as a result.
Making the commit reachable as "the current state of a branch" requires
a separate ref update — covered in full in Demo 05.

Trap: D is wrong — `git commit-tree` works entirely at the object level
and has no dependency on a working directory.

</details>

---

**Q4.** What does `git diff --cached` show?

- A) Changes in the working directory that have not been staged
- B) Changes between the staging area (index) and the last commit
- C) Changes between two commits
- D) Changes between the working directory and the last commit

<details>
<summary>Answer</summary>

**B** — `git diff --cached` (also `git diff --staged`) compares the
index against the last commit (HEAD). It shows exactly what will be
included in the next `git commit` — the pre-commit review command.

Trap: A describes `git diff` (no flags). D describes `git diff HEAD`.

</details>

---

**Q5.** You run `git rm --cached config.json` and get an error:
`"staged content different from both the file and the HEAD"`.
What is the correct explanation?

- A) `config.json` is not tracked — you need to `git add` it first
- B) The index version differs from both the working directory and HEAD, so Git refuses to silently discard the staged snapshot
- C) `git rm --cached` requires a resolvable HEAD to function
- D) The file must be committed before it can be removed from the index

<details>
<summary>Answer</summary>

**B** — `git rm --cached` has one specific refusal case: when the index
holds a version of the file that exists nowhere else — not in the working
directory (which has a newer version) and not in HEAD (which doesn't exist
or has yet another version). Removing the index entry in that state would
permanently discard the staged snapshot. Git requires `-f` to confirm you
accept that consequence.

Trap: C describes `git restore --staged`, not `git rm --cached`. D is
wrong — `git rm --cached` works in repos with no commits at all.

</details>

---

**Q6.** What is the key difference between `git add .` and `git add -A`?

- A) `git add .` stages all files; `git add -A` stages only modified files
- B) `git add -A` is recursive; `git add .` is not
- C) `git add .` stages changes in the current directory subtree; `git add -A` stages all changes across the entire repository regardless of current directory
- D) They are identical — both do the same thing

<details>
<summary>Answer</summary>

**C** — `git add .` stages from the current directory downward — files in
sibling or parent directories are not affected. `git add -A` stages
everything in the entire repository root, from wherever you run it.
Both are recursive; both stage new, modified, and deleted files within
their respective scopes. The practical difference matters most when you
run `git add .` from a subdirectory — you will only stage that subtree,
not the whole repo.

Trap: B is wrong — `git add .` is also recursive within its scope.
D is wrong — scope and deletion coverage differ between them.

</details>

---

**Q7.** When does `git add` create tree objects?

- A) Immediately — one tree per directory staged
- B) Only when `git add -A` is used
- C) Never — `git add` only creates blobs and updates the index
- D) When the staged directory contains more than one file

<details>
<summary>Answer</summary>

**C** — `git add` creates blob objects for file contents and updates
`.git/index` entries with flat file paths. Tree objects are created
later by `git write-tree` (called internally by `git commit`). The index
stores `src/utils/helpers.js` as one flat path — directory trees are
reconstructed from those paths only when `write-tree` runs.

</details>

---

**Q8.** `git write-tree` is called twice on the same repo with no index
changes between calls. What does the second call return?

- A) An error — tree objects cannot be created twice
- B) A different hash — Git appends a version counter to avoid duplicates
- C) The same hash as the first call — same content always produces the same SHA-1
- D) An empty hash — the objects already exist and are not rewritten

<details>
<summary>Answer</summary>

**C** — Git is content-addressed. Same index content produces the same
tree content produces the same SHA-1 hash. `git write-tree` is idempotent.
The second call finds all the needed tree objects already in
`.git/objects/` and returns the identical root hash. This is the same
principle that let Part 2 reach the identical tree hash two different
ways — via `mktree`+`read-tree` by hand, and via `write-tree` from a
populated index.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 8/8 | Import Anki CSV and move to Demo 05 |
| 6–7/8 | Review the wrong answers, then proceed |
| 5/8 | Re-read the relevant README section, retry quiz |
| Below 5/8 | Re-read Demo 04 and redo the walkthrough before proceeding |
````