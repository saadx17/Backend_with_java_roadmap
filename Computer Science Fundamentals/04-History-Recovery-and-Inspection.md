# History, Recovery & Inspection (log, reflog, diff, blame, stash, bisect)

> **Phase 0 — Computer Science Fundamentals → 0.4 Version Control — Git (Mastery)**
> Goal: Master inspecting and navigating Git history, undoing mistakes safely, stashing work-in-progress, recovering "lost" commits with reflog, and finding bugs with bisect.

---

## 0. The Big Picture

Git's history isn't just a record — it's a **debugging and recovery toolkit**. This note covers how to **inspect** history, **undo** mistakes, **rescue** lost work, and **hunt down** the commit that introduced a bug.

> A huge confidence boost in Git is realizing: **almost nothing is truly lost.** Committed work can almost always be recovered (via reflog).

---

## 1. Inspecting History

```bash
git log                          # full commit history
git log --oneline                # one line per commit (hash + message)
git log --oneline --graph --all  # visual branch graph ★
git log -p                       # show the diff of each commit
git log --stat                   # files changed + line counts per commit
git log -n 5                     # last 5 commits
git log --author="Alice"         # filter by author
git log --since="2 weeks ago"    # filter by date
git log -- path/to/file          # history of a specific file
git log --grep="bugfix"          # search commit messages
```

### 1.1 `git diff` — see changes
```bash
git diff                  # unstaged changes (working dir vs staging)
git diff --staged         # staged changes (staging vs last commit)
git diff main feature     # difference between two branches
git diff abc123 def456    # between two commits
git diff HEAD~3 HEAD      # last 3 commits' cumulative changes
```

### 1.2 `git show`
```bash
git show abc123           # full details + diff of a commit
git show HEAD             # the latest commit
git show abc123:file.txt  # a file's contents at a specific commit
```

---

## 2. `git blame` — Who Changed This Line?

`git blame` shows, for each line of a file, **which commit and author** last modified it — invaluable for understanding *why* code is the way it is.
```bash
git blame file.java                # annotate each line with commit/author/date
git blame -L 40,60 file.java       # only lines 40-60
```
```
abc123  (Alice  2026-05-01 14:22)  40)  int timeout = 30;
def456  (Bob    2026-06-10 09:15)  41)  retry(timeout);
```
> Use blame to find *who/when/why* a line was introduced, then `git show <hash>` to see the full commit and its message/PR context. (Not for blaming people — for understanding history!)

---

## 3. Stashing (Park Work-in-Progress)

`git stash` temporarily **shelves uncommitted changes** so you can switch contexts (e.g., handle an urgent fix) with a clean working directory, then restore them later.
```bash
git stash                    # shelve all uncommitted changes (clean working dir)
git stash push -m "wip search"  # stash with a label
git stash list               # list stashes (stash@{0}, stash@{1}, ...)
git stash pop                # restore the latest stash AND remove it from the list
git stash apply              # restore but KEEP it in the stash list
git stash apply stash@{1}    # restore a specific stash
git stash drop stash@{0}     # delete a stash
git stash -u                 # include untracked files
```

### 3.1 Typical use
```
Working on a feature (uncommitted) -> urgent bug comes in
git stash                 # park the feature work
git switch main && fix the bug && commit
git switch feature
git stash pop             # resume the feature work
```
> Stash is for **short-term** parking. For longer work, prefer committing on a branch.

---

## 4. Undoing Changes (Safely)

There are several ways to undo, depending on **what** you want to undo and **whether it's been shared**.

### 4.1 Discard uncommitted changes
```bash
git restore file.txt          # discard unstaged changes to a file (modern)
git restore .                 # discard all unstaged changes ⚠️ (can't undo)
git checkout -- file.txt      # older equivalent
git restore --staged file.txt # UNstage a file (keep the changes)
```

### 4.2 `git reset` — move the branch pointer (rewrites local history)
```bash
git reset --soft HEAD~1   # undo last commit, KEEP changes STAGED
git reset --mixed HEAD~1  # undo last commit, keep changes UNSTAGED (default)
git reset --hard HEAD~1   # undo last commit AND DISCARD changes ⚠️ destructive
```
| Mode | Commit undone? | Staging | Working dir |
|------|:--------------:|---------|-------------|
| `--soft` | ✅ | keeps changes staged | unchanged |
| `--mixed` (default) | ✅ | unstages changes | unchanged |
| `--hard` | ✅ | **discards** | **discards** ⚠️ |

> `reset` **rewrites history** → only use on **local, unshared** commits (same Golden Rule as rebase). `--hard` permanently discards uncommitted work — use with care.

### 4.3 `git revert` — undo by adding a new commit (safe for shared history)
```bash
git revert abc123         # create a NEW commit that undoes abc123's changes
```
> `revert` doesn't rewrite history — it **adds an inverse commit**. This is the **safe way to undo a commit that's already pushed/shared**.

### 4.4 reset vs revert
| | `git reset` | `git revert` |
|---|-------------|--------------|
| How it undoes | Moves the branch pointer back | Adds a new "inverse" commit |
| Rewrites history? | **Yes** | No |
| Safe on shared branches? | ❌ No | ✅ Yes |
| Use for | Local cleanup | Undoing pushed commits |

### 4.5 `git restore` vs `reset` vs `revert` (quick guide)
- Discard *uncommitted* file changes → `git restore`
- Undo *local, unpushed* commits → `git reset`
- Undo *pushed/shared* commits → `git revert`

---

## 5. `git reflog` — The Undo Safety Net (Recovery)

`git reflog` records **every** place HEAD has been — every commit, checkout, reset, rebase, merge — even ones no branch points to anymore. This is how you **recover "lost" commits**.
```bash
git reflog                     # show HEAD's movement history
# example output:
# abc123 HEAD@{0}: reset: moving to HEAD~1
# def456 HEAD@{1}: commit: Add feature   <- the "lost" commit!
git reset --hard def456        # restore to that commit
# or: git checkout def456 ; git switch -c recovered
```
> **The reflog is your safety net.** Did a `reset --hard` and lost a commit? Rebased badly? The reflog still has the old hashes (for ~90 days by default). **Committed work is almost never truly gone.**

---

## 6. `git bisect` — Find the Commit That Introduced a Bug

`git bisect` does a **binary search** through history to pinpoint exactly which commit introduced a bug — far faster than checking commits one by one (recall binary search, O(log n), Phase 0/2).
```bash
git bisect start
git bisect bad                 # current commit is broken
git bisect good v1.0           # this old commit/tag was working
# Git checks out a commit halfway between -> you TEST it:
git bisect good                # if this commit works
git bisect bad                 # if it's broken
# ... repeat; Git narrows down by half each time ...
# Git announces: "abc123 is the first bad commit"
git bisect reset               # return to where you started
```
- For *N* commits, bisect finds the culprit in ~**log₂(N)** steps (e.g., 1000 commits → ~10 tests).
- Can be **automated** with a test script: `git bisect run ./test.sh`.
> Bisect is a superpower for "it worked last week, now it's broken — which change did it?"

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `reset --hard` losing work | Recover via `git reflog` |
| `reset` on pushed commits | Use `revert` for shared history |
| Forgetting stashed work | `git stash list`; prefer branches for long work |
| "I lost a commit forever" | Check `git reflog` — it's likely still there |
| Confusing `restore`, `reset`, `revert` | restore=files, reset=local commits, revert=shared commits |
| Blaming people via `git blame` | Use it to understand history, not assign fault |
| Checking commits one-by-one for a bug | Use `git bisect` (binary search) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Debugging regressions (Phase 13):** `git bisect` + `git blame` quickly find which change broke production.
- **Safe undo in shared repos:** `git revert` to roll back a bad merge/deploy without rewriting history (Phase 10).
- **reflog confidence:** lets you experiment fearlessly, knowing you can recover.
- **`git log`/`diff`/`show`** are everyday tools for reviewing changes and understanding code evolution (Phase 6 reviews).
- **Stash** keeps your flow smooth when urgent fixes interrupt feature work.
- Clean history (next note's squashing) makes blame/bisect dramatically more effective.

---

## 9. Quick Self-Check Questions

1. How do you view a compact, visual branch graph of history?
2. What does `git diff --staged` show vs `git diff`?
3. What is `git blame` for, and how do you use it responsibly?
4. When and how do you use `git stash`?
5. What's the difference between `--soft`, `--mixed`, and `--hard` reset?
6. When must you use `git revert` instead of `git reset`?
7. What does `git reflog` record, and how does it help recover lost commits?
8. How does `git bisect` find a bug, and why is it efficient?

---

## 10. Key Terms Glossary

- **`git log` / `diff` / `show`:** view history / changes / a commit.
- **`git blame`:** show who last changed each line.
- **`git stash`:** shelve uncommitted changes temporarily.
- **`git restore`:** discard working-dir changes / unstage files.
- **`git reset` (`--soft`/`--mixed`/`--hard`):** move the branch pointer (rewrites local history).
- **`git revert`:** undo a commit by adding an inverse commit (safe for shared history).
- **`git reflog`:** log of all HEAD movements — recovery safety net.
- **`git bisect`:** binary search through history to find a bad commit.
- **HEAD~n:** n commits before HEAD.

---

*Previous topic: **Remotes & Collaboration**.*
*Next topic: **Workflow Mastery (.gitignore, tags, hooks, branching strategies, conventional commits, squashing, signed commits)**.*
