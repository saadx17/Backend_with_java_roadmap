# Remotes & Collaboration (clone, push, pull, fetch, PRs)

> **Phase 0 — Computer Science Fundamentals → 0.4 Version Control — Git (Mastery)**
> Goal: Master working with remote repositories — clone, push, pull, fetch — and the Pull Request / code review workflow that teams use every day.

---

## 0. The Big Picture

A **remote** is a version of your repository hosted elsewhere (GitHub, GitLab, Bitbucket) that you and your team sync with. Git is **distributed** (recall internals note): you work locally, then **push** to share and **pull** to receive others' work.

```
Your local repo  <--push/pull/fetch-->  Remote (origin) on GitHub  <-->  Teammates
```

> Collaboration is where Git earns its keep — multiple people building the same codebase safely.

---

## 1. Remotes

A remote is a named URL pointing to a hosted repo. The default remote is conventionally called **`origin`**.

```bash
git remote -v                              # list remotes (fetch & push URLs)
git remote add origin <url>                # link a local repo to a remote
git remote remove origin                   # remove a remote
git remote rename origin upstream          # rename
```

### 1.1 Remote-tracking branches
Git keeps **remote-tracking branches** like `origin/main` — a *local cache* of where a branch was on the remote the last time you synced. They're updated by `fetch`/`pull`, not by your local commits.
```
main           -> your local branch (you commit here)
origin/main    -> last-known state of main on the remote
```

---

## 2. Cloning

`git clone` downloads a **full copy** of a remote repository (all history) and sets up `origin` automatically:
```bash
git clone https://github.com/user/repo.git        # clone via HTTPS
git clone git@github.com:user/repo.git            # clone via SSH (recall SSH keys, 0.3)
git clone <url> my-folder                          # into a specific folder
```
After cloning you have the full history locally and can work offline.

---

## 3. Pushing (Sharing Your Commits)

`git push` uploads your local commits to the remote:
```bash
git push origin main                       # push the main branch to origin
git push -u origin feature-login           # push & set upstream (first time)
                                           # -u links local <-> remote branch
git push                                   # after -u, just this works
git push --force-with-lease                # safer forced push (see §6)
```
- `-u` (`--set-upstream`) links your local branch to the remote one, so future `git push`/`git pull` need no arguments.
- You can only push if your local branch is **up to date** with the remote — otherwise Git rejects it (pull first).

---

## 4. Fetching vs Pulling (Important Distinction)

Both bring down remote changes, but they differ in whether they **modify your working branch**.

### 4.1 `git fetch` — download, don't merge
```bash
git fetch origin            # download new commits & update origin/* refs
                            # your local branches are UNCHANGED
git log origin/main         # inspect what changed before integrating
```
> `fetch` is **safe**: it updates your knowledge of the remote (`origin/main`) without touching your work. You then decide how to integrate.

### 4.2 `git pull` — fetch + integrate
```bash
git pull origin main        # = git fetch + git merge origin/main
git pull --rebase origin main  # = git fetch + git rebase (linear history)
```
> `pull` = **fetch + merge** (by default) into your current branch. `pull --rebase` replays your local commits on top instead of creating a merge commit (recall rebase note) — many teams prefer this for cleaner history.

### 4.3 fetch vs pull
| | `git fetch` | `git pull` |
|---|-------------|------------|
| Downloads remote commits | ✅ | ✅ |
| Updates `origin/*` refs | ✅ | ✅ |
| Modifies your current branch | ❌ | ✅ (merges/rebases) |
| Safety | Always safe | Can cause conflicts/merge commits |

> **Best practice:** `fetch` then review when unsure; `pull` (often `--rebase`) for routine syncing of your own branch.

---

## 5. The Daily Collaboration Workflow

A typical day on a team:
```bash
git switch main
git pull                              # 1. get the latest main
git switch -c feature/add-search      # 2. branch for your work
# ... make changes, commit ...
git add .
git commit -m "Add search endpoint"
git pull --rebase origin main         # 3. stay current with main (resolve conflicts early)
git push -u origin feature/add-search # 4. publish your branch
# 5. open a Pull Request on GitHub/GitLab (see §6)
# 6. address review comments, push more commits
# 7. PR approved -> merged -> delete the branch
```

---

## 6. Pull Requests / Merge Requests (Code Review)

A **Pull Request (PR)** — called a **Merge Request (MR)** on GitLab — is a request to merge your branch into another (usually `main`). It's the **code review** mechanism: the heart of team collaboration and quality control.

### 6.1 The PR lifecycle
```
1. Push your feature branch to the remote
2. Open a PR: "merge feature/add-search -> main"
3. CI runs automatically (build, tests, lint)        (recall: triggers pipelines, Phase 10.3)
4. Teammates REVIEW: comment, request changes, approve
5. You push fixes; the PR updates automatically
6. Once approved + CI green -> MERGE (often squash or merge commit)
7. Delete the merged branch
```

### 6.2 Why PRs matter
| Benefit | Why |
|---------|-----|
| **Code review** | Catch bugs, share knowledge, enforce standards |
| **Discussion** | Inline comments on specific lines |
| **CI gate** | Tests/builds must pass before merge (Phase 10.3) |
| **History/traceability** | The PR documents *why* a change was made |
| **Protected main** | Prevents pushing untested/unreviewed code directly |

### 6.3 Merge strategies for PRs
| Strategy | Result |
|----------|--------|
| **Merge commit** | Preserves all branch commits + a merge commit |
| **Squash and merge** | Combines all branch commits into **one** clean commit on main |
| **Rebase and merge** | Replays commits linearly, no merge commit |
> **Squash** is popular for keeping `main`'s history clean (one commit per feature). (Squashing detailed in the history/cleanup note.)

### 6.4 Good PR practices
- Keep PRs **small and focused** (easier to review, fewer conflicts).
- Write a **clear title and description** (what + why).
- Respond to review comments; don't take feedback personally.
- Ensure **CI passes** before requesting review.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `pull` causing surprise merge commits | Use `pull --rebase`; or `fetch` then review |
| Pushing to a rejected (behind) branch | Pull/rebase first, then push |
| `git push --force` clobbering others | Use `--force-with-lease` (refuses if remote moved) |
| Confusing `fetch` and `pull` | fetch = download only; pull = download + integrate |
| Huge, unfocused PRs | Keep them small and single-purpose |
| Working directly on `main` | Branch for every change; protect `main` |
| Forgetting `-u` on first push | `git push -u origin <branch>` |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **PR-based workflow** is universal on engineering teams — your code is reviewed before merging (Phase 6, 14).
- **CI/CD (Phase 10.3)** triggers on push/PR: build + test + quality gates run automatically; merges deploy.
- **Protected branches + required reviews/CI** enforce quality on `main` (release safety).
- **SSH keys** (recall 0.3) authenticate pushes/pulls to remotes; CI uses tokens/keys to deploy.
- **`fetch`/review discipline** prevents accidental bad merges in shared repos.
- **Squash merges** keep a readable release history for changelogs and debugging (history note).

---

## 9. Quick Self-Check Questions

1. What is a remote, and what is `origin`?
2. What's a remote-tracking branch like `origin/main`?
3. What does `git clone` set up for you?
4. What does `-u` do on the first push?
5. What's the difference between `git fetch` and `git pull`? When use each?
6. What does `git pull --rebase` do, and why might a team prefer it?
7. Describe the Pull Request lifecycle and why PRs matter.
8. What are the three PR merge strategies, and when is squash useful?
9. Why use `--force-with-lease` instead of `--force`?

---

## 10. Key Terms Glossary

- **Remote:** a hosted version of the repo you sync with.
- **`origin`:** the conventional default remote name.
- **Remote-tracking branch (`origin/main`):** local cache of a remote branch's position.
- **`git clone`:** copy a remote repo locally (sets up origin).
- **`git push`:** upload local commits to a remote.
- **`git fetch`:** download remote changes without integrating.
- **`git pull`:** fetch + merge (or rebase) into your branch.
- **Upstream (`-u`):** link between a local and remote branch.
- **Pull Request (PR) / Merge Request (MR):** request + review to merge a branch.
- **Code review:** teammates inspecting changes before merge.
- **Squash / merge commit / rebase merge:** PR integration strategies.
- **`--force-with-lease`:** safer forced push that won't overwrite others' work.

---

*Previous topic: **Branching, Merging & Rebasing**.*
*Next topic: **History, Recovery & Inspection (log, reflog, diff, blame, stash, bisect)**.*
