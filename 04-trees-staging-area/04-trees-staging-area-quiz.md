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
