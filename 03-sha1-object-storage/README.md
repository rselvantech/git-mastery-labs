# Demo 03 — SHA-1 & Object Storage Deep Dive

## Overview

In Demo 02 you created blobs and trees using plumbing commands and took
it on faith that Git's SHA-1 hashing works a certain way. In this demo
you verify everything from scratch — no trust required.

You will reproduce SHA-1 hashes manually using only shell utilities,
read raw compressed object files using Python, prove the four-field
object structure by matching hashes, and understand exactly how Git
manages storage at scale through loose objects, pack files, and garbage
collection.

**What this demo covers:**

- Reproducing SHA-1 hashes from scratch — proving the header is real
- What zlib compression is and where it appears across common tools
- Reading raw zlib-compressed object files using Python
- Collision probability — why SHA-1 is safe for Git's use case
- Loose objects vs pack files — how Git manages storage at scale
- Delta compression — how pack files store file versions efficiently
- `git count-objects` — inspecting repository storage statistics
- `git gc` — what garbage collection does and when it triggers
- `git fsck` — checking object integrity and finding unreachable objects
- `git verify-pack` — reading pack file contents
- Spaced recall practice (3 questions from Demo 02 — no peeking)

---

## Prerequisites

| Requirement | Detail | Verify |
|---|---|---|
| Demo 02 complete | `src/first-project/` exists with 2 blobs + 1 tree | `find ~/git-mastery-labs/02-git-internals-object-model/src/first-project/.git/objects -type f \| wc -l` returns 3 |
| Git version | 2.28.0 or newer | `git --version` |
| Python 3 | Any recent version | `python3 --version` |
| Terminal | macOS/Linux or WSL2 | — |

---

## Concepts Covered

| Concept | What you will understand after this demo |
|---|---|
| SHA-1 reproduction | How to generate the exact same hash Git produces, from scratch |
| zlib compression | What it is, how Git uses it, and where else it appears |
| Hash collision probability | Why SHA-1 collisions are not a practical concern for Git |
| Loose objects | Individual per-object files in `.git/objects/` |
| Pack files | Compressed multi-object files Git creates for efficiency |
| Delta compression | How pack files store similar objects as diffs |
| `git count-objects` | How to measure loose object and pack file storage |
| `git gc` | What garbage collection does, when it runs, key config values |
| `git fsck` | How to check object integrity and list unreachable objects |
| `git verify-pack` | How to inspect pack file contents |

---

## Directory Structure

```
03-sha1-object-storage/
├── README.md                              # this file
├── 03-sha1-object-storage-anki.csv        # Anki flashcard deck
├── 03-sha1-object-storage-quiz.md         # standalone quiz
└── src/
    ├── first-project                      # -> ../../02-git-internals-object-model/src/first-project
    └── break-fix/                         # isolated break-fix environment
```

Create the symlink before starting the walkthrough:

```bash
cd ~/git-mastery-labs/03-sha1-object-storage/src
ln -s ../../02-git-internals-object-model/src/first-project first-project
```

> `first-project` is a symlink to Demo 02's repo. Do not delete it.
> The actual repo lives at `02-git-internals-object-model/src/first-project/`.
> All changes made here are reflected in that repo.

---

## Spaced Recall

*Answer from memory before reading anything below. No peeking at Demo 02.*

1. What is the four-field structure of every Git object?
2. A blob does not store the filename. Where is the filename stored,
   and what is the exact format of that entry?
3. You run `git read-tree <hash>` but no files appear in your working
   directory. What step is missing and what command runs it?

<details>
<summary>Reveal answers — attempt from memory first</summary>

**1.** type + space + length (bytes as decimal ASCII) + null byte `\0` + content.
The SHA-1 is computed over the entire string — not just the content.

**2.** In the tree object that points to the blob.
Format: `<mode> SP <type> SP <SHA-1> TAB <name>`
The separator between the hash and the filename is a TAB character — not a space.

**3.** `git read-tree` loads the tree into the staging area only — it does
not touch the working directory. The missing step is:
`git checkout-index -a`
which writes all staged files from `.git/index` to disk.

</details>

---

## Part 0 — zlib Compression

Before reading raw Git objects, you need to understand how they are stored.

### What is zlib?

zlib is a lossless data compression library that implements the DEFLATE
algorithm (RFC 1950/1951). "Lossless" means the original data can be
reconstructed exactly — no information is lost during compression or
decompression.

Git uses zlib to compress every object before writing it to `.git/objects/`.
This is why you cannot read object files with `cat` — the bytes on disk are
compressed and only make sense after decompression.

### Where zlib and DEFLATE appear across common tools

```
┌──────────────────────────────────────────────────────────────────┐
│  TOOL / FORMAT        │  COMPRESSION USED                        │
├───────────────────────┼──────────────────────────────────────────┤
│  Git objects          │  zlib (DEFLATE) — every blob, tree,      │
│                       │  commit, tag in .git/objects/            │
├───────────────────────┼──────────────────────────────────────────┤
│  .zip files           │  DEFLATE — same algorithm, different     │
│                       │  container format                        │
├───────────────────────┼──────────────────────────────────────────┤
│  .gz / tar.gz         │  gzip — DEFLATE with a different header  │
│                       │  tar adds the archive structure;         │
│                       │  gzip compresses the result              │
├───────────────────────┼──────────────────────────────────────────┤
│  HTTP gzip encoding   │  gzip — web servers compress responses   │
│                       │  Content-Encoding: gzip                  │
├───────────────────────┼──────────────────────────────────────────┤
│  PNG images           │  DEFLATE — pixel data is compressed      │
│                       │  before being stored in the file         │
├───────────────────────┼──────────────────────────────────────────┤
│  PDF streams          │  zlib/DEFLATE — content streams inside   │
│                       │  PDFs are often compressed               │
└──────────────────────────────────────────────────────────────────┘
```

### Why Git uses zlib for object storage

```
Text source code compresses extremely well under DEFLATE.
A typical 10,000-byte source file compresses to 3,000–4,000 bytes.
Saving 60–70% per object adds up significantly across a large repo.

Decompression is fast — much faster than the disk read itself.
The CPU cost of decompression is negligible compared to I/O savings.

The same content always compresses to the same bytes (given the same
compression level), which preserves the SHA-1 hash relationship.
```

---

## Part 1 — Reproducing SHA-1 Hashes From Scratch

In Demo 02 you saw that `git hash-object` and `shasum` produce different
hashes on the same input. You also saw that prepending the object header
to the content before piping to `shasum` produces a matching hash.

In this part you verify this completely — building the hash input by hand
and matching it byte-for-byte against what Git stored.

### Step 1 — Confirm the objects from Demo 02

```bash
cd ~/git-mastery-labs/03-sha1-object-storage/src/first-project

# List all three objects created in Demo 02
git cat-file --batch-all-objects --batch-check
# 8ab686... blob 11    ← "Hello, git\n"
# xxxxxx... blob 30    ← "Second file in git repository\n"
# xxxxxx... tree ...   ← tree pointing to both blobs
```

### Step 2 — Reproduce the first blob hash

```bash
# The blob contains "Hello, git\n" — 11 bytes including the newline
# Git hashes: "blob" + " " + "11" + "\0" + "Hello, git\n"

# "Hello, git\n" — count the bytes
printf "Hello, git\n" | wc -c
# → 11  (10 characters + 1 newline)

# Method 1: printf with null byte

# Build the full object string and hash it
printf "blob 11\0Hello, git\n" | shasum
# → 8ab686eafeb1f44702738c8b0f24f2567c36da6d  -
# Must match the hash from git cat-file --batch-all-objects output ✓

# Method 2: echo with -e flag for escape interpretation
echo -e "blob 11\0Hello, git" | shasum
# → same hash ✓

# Why the hashes match:
# shasum sees:  b l o b   1 1 \0 H e l l o ,   g i t \n
# git sees:     b l o b   1 1 \0 H e l l o ,   g i t \n
# Identical input → identical SHA-1 output
```

### Step 3 — Reproduce the second blob hash

```bash
# The second blob contains "Second file in git repository\n"
# Count the bytes first:
printf "Second file in git repository\n" | wc -c
# → 30  (29 characters + 1 newline)

# Build and hash
printf "blob 30\0Second file in git repository\n" | shasum
# → should match the hash from git cat-file --batch-all-objects output
```

### Step 4 — Why tree hashes cannot be reproduced as easily

```bash
# Tree objects use binary format internally — the SHA-1 is stored as
# raw 20-byte binary inside the tree, not as a 40-char hex string.
# Reproducing a tree hash requires binary construction — covered below.

# For now, confirm the tree contents:
TREE_HASH=$(git cat-file --batch-all-objects --batch-check | grep tree | cut -d' ' -f1)
git cat-file -p $TREE_HASH
# 100644 blob 8ab686...    file1.txt
# 100644 blob 44xxxx...    file2.txt
```

> **Why tree hashes are harder to reproduce manually:**
> Inside a tree object, each SHA-1 hash is stored as 20 raw binary bytes —
> not as the 40-character hex string you see in `git cat-file -p` output.
> To reproduce a tree hash you would need to pack each hex hash back to
> binary. This is a good illustration of why Git uses binary storage
> internally even though it displays hex to humans.

---

## Part 2 — Reading Raw Compressed Objects

Git stores every object as a zlib-compressed binary file on disk.
You cannot read them with `cat` — you get garbage output.
`git cat-file` decompresses and decodes them for you.

But you can also decompress them yourself using Python's `zlib` module.
Doing this once proves that Git has no hidden magic — just standard
compression on top of the four-field structure.

### Step 5 — Try reading a raw object with cat

```bash
# Find the path to the first blob
find .git/objects -type f | grep "^.git/objects/8a"
# .git/objects/8a/b686eafeb1f44702738c8b0f24f2567c36da6d

# Try to read it directly — output will be binary garbage
cat .git/objects/8a/b686eafeb1f44702738c8b0f24f2567c36da6d
# (unreadable binary output — zlib compressed)
```

### Step 6 — Decompress with Python zlib

```bash
# Get the full path of the blob object
OBJ_PATH=$(find .git/objects -type f | grep "8a")

# Decompress and print using Python
python3 -c "
import zlib

with open('$OBJ_PATH', 'rb') as f:   # 'rb' = read binary — required
    compressed = f.read()

decompressed = zlib.decompress(compressed)
print('Raw bytes (repr):', repr(decompressed))
print('Decoded text:    ', decompressed.decode('latin-1'))
"

# Expected output:
# Raw bytes (repr): b'blob 11\x00Hello, git\n'
# Decoded text:     blob 11\x00Hello, git
#
# \x00 is the null byte \0 — the delimiter between header and content
# This proves the four-field structure is exactly as described in Demo 02
```

### Step 7 — Parse the structure programmatically

Save as `/tmp/read-git-object.py` and run from inside `first-project/`:

```python
# Run this as a Python script for clearer output
# Save as /tmp/read-git-object.py and run: python3 /tmp/read-git-object.py

import zlib
import os
import sys

def read_git_object(repo_path, hash_prefix):
    """Find and decompress a git object by its hash prefix."""
    objects_dir = os.path.join(repo_path, '.git', 'objects')
    folder = hash_prefix[:2]
    prefix = hash_prefix[2:]

    folder_path = os.path.join(objects_dir, folder)
    if not os.path.exists(folder_path):
        print(f"No objects folder: {folder}")
        return

    for filename in os.listdir(folder_path):
        if filename.startswith(prefix):
            full_path = os.path.join(folder_path, filename)
            with open(full_path, 'rb') as f:
                raw = zlib.decompress(f.read())

            # Split at the null byte — header | content
            null_pos = raw.index(b'\x00')
            header = raw[:null_pos].decode('ascii')
            content = raw[null_pos + 1:]

            obj_type, length = header.split(' ')
            print(f"Object:  {folder}{filename}")
            print(f"Type:    {obj_type}")
            print(f"Length:  {length} bytes")
            print(f"Header:  {repr(raw[:null_pos + 1])}")
            print(f"Content: {repr(content)}")
            return

    print(f"No object found with prefix: {hash_prefix}")

# Usage: change the hash prefix to match your actual objects
read_git_object('.', '8ab6')
```

```bash
# Save and run:
python3 /tmp/read-git-object.py

# Expected output:
# Object:  8a/b686eafeb1f44702738c8b0f24f2567c36da6d
# Type:    blob
# Length:  11 bytes
# Header:  b'blob 11\x00'
# Content: b'Hello, git\n'
```

---

## Part 3 — Collision Probability

SHA-1 produces a 160-bit hash. The total number of possible hashes is:

```
2^160 = 1,461,501,637,330,902,918,203,684,832,716,283,019,655,932,542,976

≈ 1.46 × 10^48
```

### Probability of two different objects producing the same hash

Using the birthday problem approximation for collision probability:

```
Probability of collision ≈ n² / (2 × 2^160)

where n = number of objects in your repository

For a large repo with 1,000,000 objects (1 million):
  n² = 10^12
  2 × 2^160 ≈ 2.92 × 10^48

  P ≈ 10^12 / (2.92 × 10^48)
    ≈ 3.4 × 10^-37

That is: 0.00000000000000000000000000000000000034
         34 zeros after the decimal point before any non-zero digit
```

### Practical interpretation

```
If every person on Earth (8 billion people) created 1 million git objects
per second, for the entire age of the universe (13.8 billion years):

  Objects created = 8×10^9 × 10^6 × 3.15×10^7 × 1.38×10^10
                  ≈ 3.5 × 10^33

  Collision probability ≈ (3.5×10^33)² / (2 × 2^160)
                        ≈ negligible

Conclusion: SHA-1 collisions are not a practical concern for Git.
```

### The SHAttered attack (2017)

Google demonstrated a crafted SHA-1 collision using two specially constructed
PDF files. Important context:

```
What it required:
  • 9.2 × 10^18 SHA-1 computations
  • Significant computational resources
  • Carefully crafted inputs — not random files

What it means for Git:
  • Random file collisions remain astronomically unlikely
  • Crafted collisions require deliberate effort and resources
  • Git 2.13+ detects the specific SHAttered attack pattern and rejects it
  • git --hash=sha256 (available since Git 2.29) eliminates the concern

Practical conclusion:
  SHA-1 remains safe for source code repositories.
  The SHA-256 transition exists for long-term security, not urgency.
```

---

## Part 4 — Loose Objects vs Pack Files

Every object Git creates lands initially in `.git/objects/` as an
individual zlib-compressed file — one file per object.
These are called **loose objects**.

```
Current state of first-project (after Demo 02):

.git/objects/
├── 8a/b686...    # loose blob — "Hello, git\n"         (11 bytes content)
├── xx/xxxx...    # loose blob — "Second file..."        (30 bytes content)
└── xx/xxxx...    # loose tree — points to both blobs

3 loose objects — each stored as a separate zlib-compressed file
```

### The problem with loose objects at scale

```
Scenario: a repository with 10,000 commits, 500 files each
  → 10,000 × 500 = 5,000,000 potential loose object files
  → Many files change slightly between commits
  → Version 1 of app.js (10,000 bytes) and Version 2 (10,001 bytes)
     stored as two completely separate 10,000-byte compressed files
  → Massive storage waste for small changes across many versions
```

### Pack files — the solution

Git solves this with **pack files**: a single binary file that combines
many objects, using **delta compression** between similar objects.

```
┌─────────────────────────────────────────────────────────────────┐
│  PACK FILE STRUCTURE                                             │
│                                                                  │
│  .git/objects/pack/                                              │
│  ├── pack-abc123.pack   ← all the objects packed together       │
│  └── pack-abc123.idx    ← index for fast object lookup          │
│                                                                  │
│  DELTA COMPRESSION:                                              │
│  Instead of storing full copies of similar objects:             │
│                                                                  │
│  Loose:  [full blob v1: 10,000 bytes]                           │
│          [full blob v2: 10,001 bytes]  → 20,001 bytes total     │
│                                                                  │
│  Packed: [full blob v1: 10,000 bytes]                           │
│          [delta v2: "add one byte at position 5,000"]  → ~50B   │
│          Total: ~10,050 bytes  (50× smaller for small changes)  │
│                                                                  │
│  Git chooses the best "base" object and stores others as deltas │
│  Lookup via .idx file reconstructs any version on demand        │
└─────────────────────────────────────────────────────────────────┘
```

### When does packing happen?

```
Automatic triggers:
  • git push / git clone    → remote always sends objects as pack files
  • git gc (auto)           → triggers when loose object count exceeds
                              threshold (default: 6,700 loose objects)

Manual trigger:
  • git gc                  → pack all loose objects immediately
  • git pack-objects        → low-level packing (rarely used directly)

You never need to trigger packing manually for normal work.
git gc runs automatically in the background.
```

---

## Part 5 — Unreachable Objects

Not all objects in `.git/objects/` are reachable from a branch or tag.
An object is **unreachable** when no ref (branch, tag, HEAD) points to
it directly or through a chain of objects.

```
Objects become unreachable when:
  • git add creates a blob but the file is never committed
  • A commit is amended        → old commit object orphaned
  • A branch is deleted        → its commits may become unreachable
  • A rebase is performed      → old commits orphaned (new ones created)

Unreachable ≠ immediately deleted:
  Git keeps unreachable objects to allow recovery.
  They are removed by git gc after expiry windows pass:

  gc.reflogExpireUnreachable   default: 30 days
    → how long unreachable reflog entries are protected

  gc.pruneExpire               default: 2 weeks
    → minimum age before an unreachable loose object is pruned

Until pruned: any unreachable object is directly readable by hash:
  git cat-file -p <hash>

Object persistence (not reflog) is what makes this direct recovery
possible. Reachability and full recovery workflows are covered in
Demo 10 and Demo 27 respectively.
```

---

## Part 6 — Inspecting Storage with git count-objects

### Command introduction — git count-objects

```
NAME
  git count-objects — count unpacked objects and their disk usage

SYNTAX
  git count-objects [options]

OPTIONS
  (no options)   count loose objects and total size
  -v             verbose — includes pack file statistics
  -H             human-readable sizes (KB, MB, GB)

OUTPUT FIELDS (with -vH)
  count:          number of loose objects
  size:           total size of loose objects on disk
  in-pack:        number of objects stored in pack files
  packs:          number of pack files
  size-pack:      total size of all pack files
  prune-packable: loose objects that are already packed (safe to prune)
  garbage:        files in .git/objects/ that are not valid Git objects
```

### Step 8 — Check current storage statistics

```bash
cd ~/git-mastery-labs/03-sha1-object-storage/src/first-project

git count-objects -vH
# count: 3             ← 3 loose objects (2 blobs + 1 tree from Demo 02)
# size: 4.00 KiB
# in-pack: 0           ← nothing packed yet
# packs: 0
# size-pack: 0 bytes
# prune-packable: 0
# garbage: 0
```

---

## Part 7 — Garbage Collection with git gc

### Command introduction — git gc

```
NAME
  git gc — run garbage collection on the repository

SYNTAX
  git gc [options]

OPTIONS
  (no options)      pack loose objects, prune expired unreachable objects
  --prune=now       also remove all unreachable objects immediately
                    regardless of age — use with caution, not recoverable
  --aggressive      use slower but more thorough delta compression
  --auto            only run if the repo exceeds gc.auto threshold

KEY CONFIG VALUES
  gc.auto                    default: 6700
    loose object count threshold that triggers automatic gc --auto

  gc.pruneExpire             default: 2 weeks (336 hours)
    minimum age before an unreachable loose object is removed by gc

  gc.reflogExpireUnreachable default: 30 days
    how long unreachable reflog entries are protected from pruning

WHAT IT DOES
  1. Packs loose objects into pack files with delta compression
  2. Removes loose objects already present in pack files
  3. Removes unreachable objects older than gc.pruneExpire
  4. Updates the pack index (.idx file)
  5. Runs git fsck to verify integrity (in some configurations)
```

### Step 9 — Add objects and trigger gc

**Creating More Objects to Demonstrate Packing**

To demonstrate pack files meaningfully, we need more objects.
We will add some files and make commits.

### Step 9 — Add files and commit to generate more objects

```bash
cd ~/git-mastery-labs/03-sha1-object-storage/src/first-project

# Create several files with varying content
for i in $(seq 1 10); do
  echo "File number $i — content for demonstration" > file_$i.txt
done

# Add and commit
git add .
git commit -m "add ten demo files for pack demonstration"

# Check how many objects now exist
git count-objects -vH
# count: N             ← more loose objects now
find .git/objects -type f | wc -l

git cat-file --batch-all-objects --batch-check | wc -l

# Trigger garbage collection
```bash
git gc

# Expected output:
# Enumerating objects: N, done.
# Counting objects: 100% (N/N), done.
# Delta compression using up to N threads
# Compressing objects: 100% (N/N), done.
# Writing objects: 100% (N/N), done.
# Total N (delta N), reused 0 (delta 0), pack-reused 0

# Check storage after gc
git count-objects -vH
# count: 0             ← no more loose objects
# in-pack: N           ← all objects now in pack file
# packs: 1             ← one pack file created

# Verify the pack files exist
ls -lh .git/objects/pack/
# pack-abc123.pack
# pack-abc123.idx
```

---

## Part 9 — Inspecting Pack Files with git verify-pack

### Command introduction — git verify-pack

```
NAME
  git verify-pack — validate and display the contents of a pack file

SYNTAX
  git verify-pack [options] <pack-index-file>

ARGUMENT
  The .idx file — NOT the .pack file
  Located in .git/objects/pack/
  Use: find .git/objects/pack -name "*.idx"

OPTIONS
  -v    verbose — list every object with type, size, and delta depth

OUTPUT FORMAT (with -v)
  <hash> <type> <uncompressed-size> <compressed-size> <delta-depth>

  delta-depth:
    0  = stored as a full copy (the "base" object)
    N  = stored as a delta chain of depth N
         depth 1 = diff against one base object
         depth 3 = base → delta → delta → this object
```

### Step 11 — Inspect the pack file

```bash
# Find the pack index file
IDX_FILE=$(find .git/objects/pack -name "*.idx")

# List all objects in the pack
git verify-pack -v $IDX_FILE

# Output format per line:
# <hash> <type> <uncompressed-size> <compressed-size> <delta-depth>
#
# Example:
# 8ab686... blob   11  12  0   ← stored as full object (delta depth 0)
# 44xxxx... blob   31  35  0   ← stored as full object
# 3bxxxx... tree   74  78  0   ← stored as full object
# aabbcc... blob   48  30  1   ← stored as delta (depth 1 = one level deep)

# Objects with delta-depth > 0 are stored as diffs against a base object.
# Git reconstructs the full content on demand by applying the delta.
```

---

## Part 8 — Checking Integrity with git fsck

### Command introduction — git fsck

```
NAME
  git fsck — verify the integrity of the Git object database

SYNTAX
  git fsck [options]

OPTIONS
  (no options)         check all reachable objects for corruption
  --unreachable        list all objects not reachable from any ref
  --lost-found         write unreachable blobs to .git/lost-found/
                       useful for recovering orphaned content
  --no-reflogs         do not consider reflog entries as reachable
                       (shows what gc would remove if reflogs expired)

OUTPUT
  "dangling blob <hash>"   → unreachable blob (from git add never committed)
  "dangling commit <hash>" → unreachable commit (from deleted branch etc.)
  "dangling tree <hash>"   → unreachable tree
  "missing blob <hash>"    → referenced but absent — corruption indicator
  No output               → all reachable objects are intact

WHEN TO USE
  After git gc to verify nothing was lost
  To find objects from a accidentally deleted branch
  To diagnose repository corruption
```

### Step 10 — Create and find an unreachable object

Not all objects in `.git/objects/` are reachable from a branch or tag.
Objects become unreachable when:

```
• A commit is amended        → old commit object remains
• A branch is deleted        → commits on that branch become unreachable
• A rebase is performed      → old commits remain (new ones created)
• git add is run but         → blobs created even if never committed
  git reset is used to unstage
```

Unreachable objects are not immediately deleted. 
```
Unreachable objects are subject to:
  gc.reflogExpireUnreachable  (default: 30 days)  ← reflog protection window
  gc.pruneExpire              (default: 2 weeks)   ← how old before pruning
```bash

# Create a blob directly — never stage or commit it
ORPHAN=$(echo "This blob will be orphaned" | git hash-object --stdin -w)
echo "Orphaned blob hash: $ORPHAN"

# Confirm it exists
git cat-file -t $ORPHAN
# blob

# List unreachable objects — our new blob should appear
git fsck --unreachable
# unreachable blob xxxxxx...   ← our orphaned blob appears here

# Directly recover it by hash (as long as gc has not pruned it)
git cat-file -p $ORPHAN
# This blob will be orphaned
```



## CLI Verification Toolkit

```bash
# ─────────────────────────────────────────────────────
# STORAGE INSPECTION
# ─────────────────────────────────────────────────────
git count-objects -vH              # loose + pack statistics
find .git/objects -type f          # list all loose object files
ls -lh .git/objects/pack/          # list pack files

# ─────────────────────────────────────────────────────
# PACK FILE INSPECTION
# ─────────────────────────────────────────────────────
IDX=$(find .git/objects/pack -name "*.idx")
git verify-pack -v $IDX            # list all packed objects with depth

# ─────────────────────────────────────────────────────
# OBJECT INTEGRITY AND REACHABILITY
# ─────────────────────────────────────────────────────
git fsck                           # check all reachable objects
git fsck --unreachable             # list unreachable objects
git fsck --lost-found              # write orphaned blobs to .git/lost-found/

# ─────────────────────────────────────────────────────
# GARBAGE COLLECTION
# ─────────────────────────────────────────────────────
git gc                             # pack loose, prune expired unreachables
git gc --prune=now                 # also remove unreachables immediately
                                   # CAUTION: not recoverable
```

---

## Key Takeaways

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  1. SHA-1 hash reproduction proves the four-field structure          │
│     printf "blob N\0content\n" | shasum = git hash-object output     │
│     No trust required — every hash is independently verifiable       │
│                                                                      │
│  2. Git objects are zlib-compressed binary files on disk             │
│     cat produces garbage — use git cat-file -p or Python zlib        │
│     Decompressed content is always: type SP length NUL content       │
│                                                                      │
│  3. SHA-1 collisions are not a practical concern                     │
│     2^160 possible hashes. SHAttered attack requires crafted input   │
│     and enormous resources. Git 2.13+ detects it. SHA-256 available. │
│                                                                      │
│  4. Loose objects → pack files over time                             │
│     New objects start as individual loose files                      │
│     git gc packs them with delta compression — significant savings   │
│     Delta depth 0 = full copy; depth N = chain of N diffs           │
│                                                                      │
│  5. Unreachable objects persist until gc expiry windows pass         │
│     gc.pruneExpire: 2 weeks (default)                                │
│     gc.reflogExpireUnreachable: 30 days (default)                   │
│     Until then: git cat-file -p <hash> recovers any orphaned object  │
│     Reachability covered in Demo 10. Recovery workflows in Demo 27.  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Quick reference — commands used in this demo

| Command | What it does |
|---|---|
| `printf "blob N\0content\n" \| shasum` | Reproduce a Git blob hash manually |
| `python3 -c "import zlib..."` | Decompress a raw Git object file |
| `git count-objects -vH` | Show loose object and pack file statistics |
| `git gc` | Pack loose objects and prune expired unreachable objects |
| `git gc --prune=now` | Also remove unreachable objects immediately |
| `git fsck` | Check all reachable objects for corruption |
| `git fsck --unreachable` | List objects not reachable from any ref |
| `git fsck --lost-found` | Write orphaned blobs to .git/lost-found/ |
| `git verify-pack -v <idx>` | List all objects in a pack file with type and delta depth |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `printf "blob N\0..."` hash does not match | Wrong byte count N | Verify: `printf "content\n" \| wc -c` — count includes the newline |
| `python3` zlib error: `not enough data` | File opened in text mode | Open with `'rb'` (read binary) — required for compressed data |
| `python3` zlib error: `invalid header` | Wrong object file path | Verify path with `find .git/objects -type f \| grep <prefix>` |
| `git gc` shows nothing to pack | Below gc.auto threshold | Run `git gc` explicitly — it always runs regardless of threshold |
| `git verify-pack` — file not found | Argument is .pack not .idx | Use the `.idx` file: `find .git/objects/pack -name "*.idx"` |
| `git fsck --unreachable` shows nothing | All objects reachable or gc already ran | Expected in a clean committed repo |

---

## Break-Fix Scenario

```bash
cd ~/git-mastery-labs/03-sha1-object-storage/src/break-fix
git init
echo "alpha content" > alpha.txt
echo "beta content"  > beta.txt
git add .
git commit -m "initial commit"
# Make more changes to create some loose objects
echo "updated alpha" > alpha.txt
git add alpha.txt
git commit -m "update alpha"
```

Run each broken command manually. Diagnose from the error output alone.
Fix and re-run before moving to the next.

```bash
# Command 1 — decompress a git object using Python
OBJ=$(find .git/objects -type f | head -1)
python3 -c "
import zlib
with open('$OBJ', 'r') as f:
    print(zlib.decompress(f.read()))
"

# Command 2 — inspect storage verbosely with human-readable sizes
git count-objects -vh

# Command 3 — inspect the pack index after running gc
git gc
git verify-pack -v .git/objects/pack/pack.idx
```

<details>
<summary>Reveal answers — attempt diagnosis first</summary>

**Error 1 — file opened in text mode `'r'` instead of binary `'rb'`**
zlib requires binary data. Opening in text mode (`'r'`) causes Python to
interpret bytes as text, inserting platform line-ending translations and
potentially corrupting the compressed data stream.
Python raises: `zlib.error: Error -3 while decompressing data: invalid block type`
or similar.
Fix: `open('$OBJ', 'rb')` — the `b` flag is mandatory for binary files.

**Error 2 — `-h` (lowercase) instead of `-H` (uppercase)**
`git count-objects` uses `-H` (uppercase) for human-readable sizes.
`-h` is not a valid flag for this command.
Git returns: `error: unknown switch 'h'`
Fix: `git count-objects -vH`

**Error 3 — wrong pack index filename**
After `git gc` the pack index file has a content-derived hash name like
`pack-abc123def456....idx` — not `pack.idx`. There is no file named `pack.idx`.
Git returns: `error: cannot open .git/objects/pack/pack.idx`
Fix: find the real filename first, then pass it:
```bash
IDX=$(find .git/objects/pack -name "*.idx")
git verify-pack -v $IDX
```

</details>

---

## What's Next

**Demo 04 — Trees, Staging Area & Working Directory**

```
✓ Build nested directory structures with trees pointing to trees
✓ Understand structural sharing — only changed objects get new hashes
✓ Use git add and observe exactly what it does to .git/index
✓ Use git diff and git diff --cached to compare areas
✓ Use git write-tree to create a tree from the current index
```

---

## References

| Topic | Resource |
|---|---|
| Git pack format | https://git-scm.com/docs/pack-format |
| git-gc documentation | https://git-scm.com/docs/git-gc |
| git-count-objects | https://git-scm.com/docs/git-count-objects |
| git-verify-pack | https://git-scm.com/docs/git-verify-pack |
| git-fsck | https://git-scm.com/docs/git-fsck |
| SHAttered attack | https://shattered.io |
| SHA-256 transition | https://git-scm.com/docs/hash-function-transition |
| Pro Git — packfiles | https://git-scm.com/book/en/v2/Git-Internals-Packfiles |
| zlib / RFC 1950 | https://datatracker.ietf.org/doc/html/rfc1950 |

---

## Appendix — Anki Cards

**03-sha1-object-storage-anki.csv:**

```
#deck:git-mastery-labs::Section 1 - Foundations::03-sha1-object-storage
#separator:Comma
#columns:Front,Back,Tags
"You want to reproduce the SHA-1 hash Git produces for a file containing 'Hello, git\n' (11 bytes). What exact command?","printf 'blob 11\0Hello, git\n' | shasum — Git prepends the header (type SP length NUL) before hashing. shasum on raw content alone gives a different result.","demo03 sha1 internals"
"You run cat on a file in .git/objects/ and get binary garbage. Why, and how do you read it correctly?","Git stores all objects as zlib-compressed binary (DEFLATE algorithm). cat cannot decompress it. Use git cat-file -p <hash> to read any object, or Python's zlib.decompress() to see raw bytes. Always open the file in binary mode ('rb').","demo03 storage compression zlib"
"What is a loose object in Git?","An individual zlib-compressed object file stored at .git/objects/<first2>/<remaining38>. Every new object starts as a loose file. Packed into pack files by git gc or during push/clone operations.","demo03 storage loose-objects"
"What is a pack file and what two files make up a pack?","A pack file combines many objects into one using delta compression — solving the storage problem of thousands of loose files. Two files: pack-<hash>.pack (the objects) and pack-<hash>.idx (the index for fast lookup).","demo03 storage pack-files"
"What is delta compression in a Git pack file?","One full 'base' object is stored. Similar objects are stored as deltas (diffs against the base). git verify-pack -v shows delta depth: 0 = full copy, N = chain of N diffs. Significantly reduces storage for incremental file changes.","demo03 storage delta compression"
"What does git count-objects -vH show?","Verbose human-readable storage statistics: count (loose objects), size (loose size), in-pack (objects in pack files), packs (pack file count), size-pack (total pack size), prune-packable (loose objects already packed).","demo03 commands count-objects"
"What triggers automatic garbage collection in Git?","git gc --auto runs in the background when loose object count exceeds gc.auto threshold (default: ~6,700). Also run manually with git gc. Push/clone exchange pack files but do NOT trigger local gc.","demo03 gc storage"
"An unreachable object exists in .git/objects/. What determines when git gc removes it?","Two config values: gc.pruneExpire (default: 2 weeks) — minimum age before a loose unreachable object is pruned. gc.reflogExpireUnreachable (default: 30 days) — how long unreachable reflog entries are protected. Until both windows expire, the object is recoverable via git cat-file -p <hash>.","demo03 gc unreachable config"
"What command lists all objects in a pack file with their type, size, and delta depth?","git verify-pack -v <path-to-.idx-file> — argument is the .idx file, NOT the .pack file. Delta depth 0 = stored as full object. Higher = delta chain depth.","demo03 commands verify-pack"
"What does git fsck --unreachable do?","Lists all objects in the database not reachable from any branch, tag, or ref — shown as 'dangling blob/commit/tree <hash>'. These are candidates for eventual removal by git gc. Can also be used to find orphaned blobs from git hash-object -w that were never committed.","demo03 commands fsck"
"You open a Git object file with Python but get a zlib error. What is the most likely cause?","The file was opened in text mode ('r') instead of binary mode ('rb'). zlib.decompress() requires raw bytes. Always use open(path, 'rb') for Git object files.","demo03 storage zlib python"
"Why does zlib compression work particularly well for Git's use case?","Source code is text — highly repetitive, compresses 60-70% under DEFLATE. Decompression is fast (CPU cost negligible vs I/O savings). Same content always compresses identically at the same level, preserving the SHA-1 hash relationship.","demo03 storage compression rationale"
```

---

## Appendix — Quiz

**03-sha1-object-storage-quiz.md:**

```
# Quiz — Demo 03: SHA-1 & Object Storage Deep Dive

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 04.

---

**Q1.** You want to reproduce the SHA-1 hash Git produces for a blob
containing "hello\n". You count: `printf "hello\n" | wc -c` → 6.
Which command produces the matching hash?

- A) `echo "hello" | shasum`
- B) `printf "blob 6\0hello\n" | shasum`
- C) `printf "blob 5\0hello\n" | shasum`
- D) `printf "hello\n" | shasum`

<details>
<summary>Answer</summary>

**B** — Git hashes: type SP length NUL content. "hello\n" is 6 bytes.
Header: `blob 6\0`. shasum on raw content (A, D) skips the header.

Trap: C uses length 5 — omitting the newline byte. Always count
the actual bytes: `printf "hello\n" | wc -c` → 6, not 5.

</details>

---

**Q2.** You open a Git object file with Python: `open(path, 'r')`.
`zlib.decompress()` raises an error. What is wrong?

- A) zlib is not available in Python's standard library
- B) The file must be opened in binary mode: `open(path, 'rb')`
- C) Git object files cannot be read by Python — use git cat-file instead
- D) The object is corrupted and needs to be regenerated

<details>
<summary>Answer</summary>

**B** — zlib.decompress() requires raw bytes. Text mode ('r') applies
platform-specific encoding and line-ending translation, corrupting the
compressed byte stream. Always use 'rb' for binary files.

Trap: A is wrong — zlib is part of Python's standard library, no install needed.

</details>

---

**Q3.** After running `git gc`, `git count-objects -vH` shows
`count: 0` and `in-pack: 47`. What is the correct interpretation?

- A) All 47 objects were deleted — the repo is empty
- B) All 47 objects moved from loose files into a pack file — repo is healthy
- C) 47 objects are corrupted — run git fsck to repair
- D) git gc failed — objects should still be in loose form

<details>
<summary>Answer</summary>

**B** — `count: 0` means no loose objects remain.
`in-pack: 47` means all 47 objects are in pack files.
This is the expected healthy outcome of a successful `git gc`.

Trap: A is wrong — `in-pack: 47` confirms the objects exist in packed form,
not that they were deleted.

</details>

---

**Q4.** `git verify-pack -v` shows an object with delta depth 2.
What does this mean for storage and retrieval?

- A) The object is stored as a full copy with 2 backup replicas
- B) The object is stored as a diff applied on top of 2 intermediate objects
- C) The object is corrupted at depth 2 and needs repair
- D) The object was compressed twice — once by Git and once by the OS

<details>
<summary>Answer</summary>

**B** — Delta depth 2 means: base object → delta 1 → this object (depth 2).
Git reconstructs the full content by applying 2 deltas in sequence.
Depth 0 = stored as a full copy (no delta chain).

Trap: A is wrong — Git does not store backup replicas within a pack.

</details>

---

**Q5.** You run `git add secret.txt` then immediately `git rm --cached secret.txt`
without committing. You then run `git gc --prune=now`.
Is the blob for secret.txt recoverable after gc?

- A) Yes — git rm --cached keeps the blob in .git/objects/ indefinitely
- B) No — git gc --prune=now removes all unreachable objects immediately
- C) Yes — the blob is still referenced by the staging area
- D) No — git rm --cached deletes the blob from .git/objects/

<details>
<summary>Answer</summary>

**B** — `git add` creates the blob and stages it. `git rm --cached` removes
the staging area entry but leaves the blob in .git/objects/ as an unreachable
object. `git gc --prune=now` removes ALL unreachable objects immediately,
regardless of age. The blob is gone after this command.

Trap: C is wrong — `git rm --cached` removes the index entry, so the blob
is no longer referenced by the staging area.
D is wrong — `git rm --cached` does not touch .git/objects/.
A is wrong — without --prune=now, the blob survives the default expiry
windows (gc.pruneExpire: 2 weeks), but --prune=now bypasses those windows.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 5/5 | Import Anki CSV and move to Demo 04 |
| 4/5 | Review the wrong answer, then proceed |
| 3/5 | Re-read the relevant README section, retry quiz |
| Below 3/5 | Re-read Demo 03 and redo the walkthrough before proceeding |
```