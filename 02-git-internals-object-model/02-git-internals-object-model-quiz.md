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

**Q5.** You delete `file1.txt` from your working directory.
What happens to its blob in `.git/objects/`?

- A) The blob is automatically deleted when the file is deleted from disk
- B) The blob remains — Git objects are never deleted automatically
- C) The blob is immediately marked as orphaned and deleted on next gc
- D) Git detects the deletion and restores the file automatically

<details>
<summary>Answer</summary>

**B** — Deleting a file from the working directory has no effect on
`.git/objects/`. The blob remains until `git gc` runs after the reflog
expiry period (90 days by default). Until then it is fully recoverable
with `git cat-file -p <hash>`.

Trap: C is partially correct in timing but wrong about "immediately" —
the object becomes unreachable but is not queued or deleted until gc runs.
D describes the opposite direction — recovery is manual, not automatic.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 5/5 | Import Anki CSV and move to Demo 03 |
| 4/5 | Review the wrong answer, then proceed |
| 3/5 | Re-read the relevant README section, retry quiz |
| Below 3/5 | Re-read Demo 02 and redo the walkthrough before proceeding |
