# Branching, Merging & Rebasing

> **Phase 0 — Computer Science Fundamentals → 0.4 Version Control — Git (Mastery)**
> Goal: Master branching and integrating work — creating/switching branches, merging (fast-forward vs 3-way), resolving merge conflicts, rebasing, and cherry-picking.

---

## 0. The Big Picture

A **branch** is an independent line of development — a movable pointer to a commit (recall internals note). Branching lets you work on features, fixes, and experiments **in isolation** without affecting the main code, then **integrate** them back via merge or rebase.

```
main:     A---B---C---------F   (F = merge of feature)
                   \       /
feature:            D---E
```

> Cheap, fast branching is Git's superpower — and the foundation of all team workflows (next note).

---

## 1. Working with Branches

```bash
git branch                     # list branches (* = current)
git branch feature-login       # create a branch (doesn't switch to it)
git switch feature-login       # switch to it (modern command)
git switch -c feature-login    # create AND switch in one step
git checkout -b feature-login  # older equivalent of -c
git switch main                # go back to main
git branch -d feature-login    # delete a (merged) branch
git branch -D feature-login    # force-delete (unmerged) — careful
git branch -m old new          # rename a branch
```
> `git switch`/`git restore` (newer) split the overloaded `git checkout` into clearer commands. `checkout` still works but is being superseded.

### 1.1 What happens when you branch
Creating a branch just **adds a new pointer** to the current commit — no files are copied. Switching branches updates your working directory to match that branch's commit and moves **HEAD**.
```
Before:  HEAD -> main -> C
After `git switch -c feature`:
         main -> C
         HEAD -> feature -> C   (both point to C; they diverge as you commit)
```

---

## 2. Merging

**Merging** integrates changes from one branch into another. You switch to the **target** branch and merge the **source** in:
```bash
git switch main
git merge feature-login        # merge feature-login INTO main
```

### 2.1 Fast-forward merge
If the target branch hasn't diverged (no new commits since the branch was created), Git just **moves the pointer forward** — no merge commit needed:
```
Before:  main -> A      feature -> A---B---C
After FF: main -> C  (just moved the pointer; linear history)
```

### 2.2 Three-way merge (merge commit)
If **both** branches have new commits (they diverged), Git creates a **merge commit** with **two parents**, combining both lines:
```
main:     A---B---E
                   \
feature:   \--C---D \
                    F  <- merge commit (parents: E and D)
main -> F
```
```bash
git merge feature              # creates a merge commit if branches diverged
git merge --no-ff feature      # FORCE a merge commit even if FF is possible
                               # (keeps the feature's history visible as a group)
```

### 2.3 Fast-forward vs no-ff
| | Fast-forward | Merge commit (--no-ff) |
|---|--------------|------------------------|
| History | Linear (clean) | Shows the branch as a distinct group |
| Merge commit | None | Yes (extra commit) |
| Traceability | Less | More (clear feature boundaries) |
| Common in | Simple/solo work | Team workflows (PR merges) |

---

## 3. Merge Conflicts

A **conflict** happens when the same lines were changed differently in both branches — Git can't decide automatically, so **you** resolve it.

### 3.1 What a conflict looks like
Git marks the conflicting region in the file:
```
<<<<<<< HEAD
int timeout = 30;          // your current branch's version
=======
int timeout = 60;          // the incoming branch's version
>>>>>>> feature-login
```

### 3.2 Resolving a conflict
```bash
git merge feature           # -> CONFLICT reported
git status                  # shows which files conflict ("both modified")
# 1. Open each conflicted file, edit to the correct final result,
#    and REMOVE the <<<<<<<, =======, >>>>>>> markers.
git add conflicted-file.java   # 2. mark it resolved by staging
git commit                  # 3. complete the merge (or `git merge --continue`)

git merge --abort           # bail out and return to the pre-merge state
```
| Step | Action |
|------|--------|
| 1 | Edit files to the desired final content, remove conflict markers |
| 2 | `git add` each resolved file |
| 3 | `git commit` (or `--continue`) to finish |

> **Tip:** keep branches short-lived and pull/rebase from main frequently to minimize conflicts. Merge/diff tools (IntelliJ, VS Code) make resolution much easier than editing markers by hand.

---

## 4. Rebasing

**Rebasing** moves (or "replays") your branch's commits onto the tip of another branch — producing a **linear history** instead of a merge commit.

```bash
git switch feature
git rebase main             # replay feature's commits on top of the latest main
```
```
Before:  main:    A---B---E
         feature: A---C---D

After `git rebase main` (on feature):
         feature: A---B---E---C'---D'   (C,D rewritten as C',D' on top of E)
```

### 4.1 Merge vs Rebase
| | **Merge** | **Rebase** |
|---|-----------|------------|
| History | Preserves true history (merge commit) | Rewrites into a linear history |
| Result | Non-linear (branches visible) | Clean, straight line |
| Commit hashes | Unchanged | **Changed** (commits are rewritten) |
| Safety | Safe on shared branches | **Dangerous** on shared branches |
| Best for | Integrating shared/public branches | Tidying *your own local* branch before sharing |

### 4.2 The Golden Rule of Rebasing
> ⚠️ **Never rebase commits that have been pushed and that others may have based work on.** Rebasing rewrites history (new hashes); if others have the old commits, you create divergence and chaos. **Rebase only your own local, unpushed commits.**

### 4.3 When to use which
- **Rebase** your feature branch onto main to keep it up to date and linear *before* opening a PR (local, private work).
- **Merge** to integrate a finished feature into main (often via a PR), preserving the integration point.
- Many teams use: *rebase locally to stay current, merge (PR) to integrate.*

### 4.4 Interactive rebase (preview — full detail in history/cleanup note)
```bash
git rebase -i HEAD~3        # interactively edit the last 3 commits
```
Lets you **squash**, reorder, edit, or drop commits to craft a clean history before sharing (covered in the workflow note).

---

## 5. Cherry-Pick

**Cherry-picking** applies a **single commit** from one branch onto another — without merging the whole branch. Useful for hotfixes or grabbing one specific change.
```bash
git switch main
git cherry-pick abc123      # apply commit abc123 onto main
git cherry-pick A B C       # apply several specific commits
```
```
hotfix:  X---Y---Z
main:    A---B---C
git cherry-pick Y  ->  main: A---B---C---Y'   (just Y, copied as Y')
```
> Common use: a bug fix made on one branch needs to go into a release branch too — cherry-pick just that commit. The cherry-picked commit gets a **new hash** (it's a copy).

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Rebasing shared/pushed commits | Never — only rebase local, unpushed work |
| Leaving conflict markers in files | Remove `<<<`, `===`, `>>>` and verify the result |
| Force-deleting unmerged branches (`-D`) | Make sure work is saved/merged first |
| Long-lived branches → huge conflicts | Keep branches short; sync with main often |
| Confusing merge direction | You merge the source *into* the currently checked-out branch |
| Expecting rebase to keep hashes | Rebase rewrites commits (new hashes) |
| Panicking during a bad merge/rebase | `git merge --abort` / `git rebase --abort` |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Feature branches + PRs** are the standard team workflow (next note, Phase 6 code reviews).
- **Merge conflicts** are a daily reality on teams — resolving them confidently is essential.
- **Rebase to stay current** keeps feature branches clean before review; **merge** integrates them.
- **Cherry-pick** is how hotfixes get applied across release branches (Phase 10 releases).
- **Clean history** (rebase/squash) makes `git bisect`, `git blame`, and code archaeology far easier (history note).
- **CI/CD** runs on branches and PRs (Phase 10.3) — branching discipline keeps pipelines green.

---

## 8. Quick Self-Check Questions

1. What is a branch, and what happens (internally) when you create one?
2. What's the difference between a fast-forward and a three-way merge?
3. When does `--no-ff` matter?
4. What causes a merge conflict, and what are the steps to resolve one?
5. How do you abort an in-progress merge or rebase?
6. What does rebase do, and how does it differ from merge?
7. State the Golden Rule of Rebasing and why it exists.
8. What is cherry-pick, and when would you use it?

---

## 9. Key Terms Glossary

- **Branch:** an independent line of development (a movable pointer).
- **`git switch` / `checkout`:** change the current branch.
- **Merge:** integrate one branch's changes into another.
- **Fast-forward:** moving a branch pointer forward (no merge commit).
- **Three-way merge / merge commit:** combining diverged branches (two parents).
- **`--no-ff`:** force a merge commit.
- **Merge conflict:** overlapping changes Git can't auto-resolve.
- **Conflict markers:** `<<<<<<<`, `=======`, `>>>>>>>`.
- **Rebase:** replay commits onto another base (linear history; rewrites hashes).
- **Golden Rule of Rebasing:** don't rebase shared/pushed history.
- **Interactive rebase:** edit/squash/reorder commits.
- **Cherry-pick:** copy a single commit onto another branch.

---

*Previous topic: **Git Fundamentals & Internals**.*
*Next topic: **Remotes & Collaboration (clone, push, pull, fetch, PRs)**.*
