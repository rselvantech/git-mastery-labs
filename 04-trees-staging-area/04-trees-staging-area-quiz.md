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
