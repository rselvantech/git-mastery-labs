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
