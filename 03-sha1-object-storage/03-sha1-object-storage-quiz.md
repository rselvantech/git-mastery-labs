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
