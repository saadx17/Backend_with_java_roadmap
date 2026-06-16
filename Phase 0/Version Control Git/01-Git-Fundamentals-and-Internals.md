# Git Fundamentals & Internals

> **Phase 0 — Computer Science Fundamentals → 0.4 Version Control — Git (Mastery)**
> Goal: Understand what version control is and why it matters, how Git works internally (blobs, trees, commits, refs), and the core local workflow (init, add, commit, the three areas).

---

## 0. The Big Picture

**Version control** is a system that records changes to files over time so you can recall specific versions, collaborate, and never lose work. **Git** is the dominant version control system — distributed, fast, and the foundation of modern software collaboration (GitHub, GitLab, Bitbucket).

```
Edit files  ->  stage changes  ->  commit (snapshot)  ->  push to share
```

> As a backend engineer, **Git is non-negotiable.** Every team uses it daily for code, reviews, branching, releases, and history. This section takes you from basics to mastery.

---

## 1. What Is Version Control & Why

| Without version control | With Git |
|-------------------------|----------|
| `report_final_v2_REALLY_final.docx` | Clean history of every change |
| Overwriting each other's work | Merge changes from many people |
| "What did I change last week?" | `git log`, `git diff`, `git blame` |
| No way to undo a bad change | Revert to any past commit |
| Fear of experimenting | Branches: experiment safely |

### 1.1 Centralized vs Distributed
- **Centralized (old, e.g., SVN):** one central server holds history; you check files out/in.
- **Distributed (Git):** **every clone is a full copy** of the entire history. You can commit, branch, and view history **offline**; you sync with others by pushing/pulling.

> Git being distributed means your local repo is complete — you don't need the network to work, only to share.

---

## 2. Git's Three Areas (The Core Mental Model)

This is the single most important concept for using Git correctly:

```
 Working Directory   ->   Staging Area (Index)   ->   Repository (.git)
  (your edited files)      (changes you've marked)     (committed snapshots)
        |                          |                          |
     git add ------------------->  |                          |
                              git commit -------------------> |
```

| Area | What it is | Command to move into it |
|------|-----------|-------------------------|
| **Working Directory** | Your actual files on disk (where you edit) | (you edit them) |
| **Staging Area (Index)** | A "draft" of the next commit — changes you've selected | `git add` |
| **Repository** | Permanent committed history (in `.git/`) | `git commit` |

### 2.1 Why a staging area?
The staging area lets you **craft a commit deliberately** — choose exactly which changes go in (even parts of a file with `git add -p`), so each commit is focused and meaningful, rather than dumping everything at once.

---

## 3. The Core Local Workflow

```bash
git init                      # 1. create a new repo (makes the .git/ folder)
# ... edit files ...
git status                    # 2. see what's changed / staged
git add file.txt              # 3. stage a specific file
git add .                     # 3. stage everything changed
git commit -m "Add feature"   # 4. snapshot the staged changes into history
git log                       # 5. view the commit history
```

### 3.1 First-time setup (identity)
```bash
git config --global user.name  "Alice Dev"
git config --global user.email "alice@example.com"
git config --global init.defaultBranch main   # use 'main' as default branch
```

### 3.2 `git status` — your compass
Run it constantly. It shows: which branch you're on, staged changes (green), unstaged changes (red), and untracked files.

### 3.3 File states in Git
```
Untracked  ->  Staged  ->  Committed  ->  Modified  ->  Staged  -> ...
   (new)     (git add)    (git commit)   (you edit)   (git add)
```
| State | Meaning |
|-------|---------|
| **Untracked** | New file Git isn't tracking yet |
| **Modified** | Tracked file changed but not staged |
| **Staged** | Marked for the next commit |
| **Committed** | Safely stored in the repository |

---

## 4. Git Internals (How It Actually Works)

Understanding Git's internals demystifies *everything* about it. Git is, at its core, a **content-addressable key-value store**: it stores **objects** identified by the **SHA-1 hash** of their content.

### 4.1 The four object types
```
Commit  ->  points to a  Tree  ->  points to  Blobs (file contents) and sub-Trees
```
| Object | Stores | Analogy |
|--------|--------|---------|
| **Blob** | The **contents** of a file (no name, no metadata) | A file's data |
| **Tree** | A directory listing: names + permissions → blobs/sub-trees | A folder |
| **Commit** | A snapshot: points to one tree + parent commit(s) + author + message | A save point |
| **Tag** | A named pointer to a specific commit (annotated tags are objects) | A label |

### 4.2 How a commit is structured
```
Commit abc123
├── tree         -> 9f8e7d  (the root directory snapshot)
├── parent       -> def456  (the previous commit; merges have 2+ parents)
├── author       -> Alice <alice@...>  date
├── committer    -> Alice <alice@...>  date
└── message      -> "Add login feature"

Tree 9f8e7d
├── src/    (sub-tree)
└── README.md -> blob 1a2b3c  (file content)
```
- Each object has a unique **SHA-1 hash** (40 hex chars, e.g., `abc123...`) computed from its content. Identical content → identical hash → Git **stores it once** (deduplication).
- A **commit snapshots the whole project** (via its tree), not just diffs — diffs are *computed* between snapshots when you ask.

### 4.3 Refs — human-readable pointers
You don't memorize hashes; Git uses **refs** (references) — named pointers to commits:
| Ref | Points to |
|-----|-----------|
| **Branch** (e.g., `main`) | The latest commit on that line of work (a *movable* pointer) |
| **HEAD** | The commit/branch you currently have checked out ("you are here") |
| **Tag** | A *fixed* pointer to a specific commit (e.g., a release) |
| **Remote refs** (`origin/main`) | The last-known position of a branch on the remote |

```
HEAD -> main -> commit abc123 -> commit def456 -> ... (parent chain)
```
> A **branch is just a lightweight, movable pointer** to a commit — which is why branching in Git is instant and cheap (covered next note).

### 4.4 The .git directory
Everything lives in the hidden `.git/` folder at the repo root: objects, refs, the index (staging area), config, and HEAD. Deleting `.git/` removes all version history (the files remain).

---

## 5. Viewing & Inspecting History (essentials)

```bash
git log                        # full history
git log --oneline             # compact: one line per commit
git log --oneline --graph --all  # visualize branches as a graph
git log -p                    # show diffs for each commit
git show <commit>             # details + diff of a specific commit
git diff                      # unstaged changes (working dir vs staged)
git diff --staged            # staged changes (vs last commit)
```
(More history tools — blame, reflog, bisect — in a later note.)

---

## 6. .gitignore (preview — full detail later)

A `.gitignore` file lists patterns Git should **not track** (build artifacts, secrets, IDE files):
```gitignore
target/          # Maven build output
*.class          # compiled Java
.env             # secrets — NEVER commit
.idea/           # IntelliJ files
```
> Recall Phase 1: `target/`, `.class` files are build artifacts — they're regenerated and shouldn't be in version control. (Covered fully in the workflow note.)

---

## 7. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "Git and GitHub are the same" | Git = the tool; GitHub = a hosting service for Git repos |
| "Commits store diffs" | Commits store **snapshots** (trees); diffs are computed |
| Forgetting `git add` before commit | Only **staged** changes are committed |
| "A branch is a copy of files" | It's just a **movable pointer** to a commit |
| Committing secrets/build artifacts | Use `.gitignore`; secrets in history are very hard to remove |
| Not running `git status` | It's your compass — run it constantly |
| Deleting `.git/` to "clean up" | That erases all history |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Every project** is a Git repo; you commit code, configs, migrations, and CI/CD definitions.
- **`.gitignore`** keeps `target/`, secrets, and IDE files out of the repo (Phase 1, 3, 15).
- **Branching & PRs** (next notes) are how teams collaborate and review code (Phase 6 code reviews).
- **CI/CD (Phase 10.3)** triggers on Git events (push, PR) — Git is the entry point of the pipeline.
- **Understanding internals** makes advanced operations (rebase, reflog recovery, bisect) intuitive instead of scary.

---

## 9. Quick Self-Check Questions

1. What problem does version control solve? What does "distributed" mean for Git?
2. Name Git's three areas and the command that moves changes between each.
3. Why does a staging area exist?
4. What are the four Git object types, and what does each store?
5. Does a commit store a diff or a snapshot? How are diffs produced?
6. What is a ref? What are HEAD and a branch?
7. Why is creating a branch in Git cheap?
8. What's the difference between Git and GitHub?

---

## 10. Key Terms Glossary

- **Version control:** system tracking changes to files over time.
- **Distributed VCS:** every clone has the full history (Git).
- **Working directory / staging area (index) / repository:** Git's three areas.
- **`git add` / `git commit`:** stage changes / snapshot them.
- **Blob / tree / commit / tag:** Git's object types (content / directory / snapshot / label).
- **SHA-1 hash:** content-derived object identifier.
- **Snapshot:** a commit's full picture of the project (via a tree).
- **Ref:** a named pointer to a commit.
- **Branch:** a movable pointer to the latest commit on a line of work.
- **HEAD:** pointer to the currently checked-out commit/branch.
- **`.git/`:** the directory holding all repository data.
- **`.gitignore`:** patterns of files Git should not track.

---

*This is the first note of **Section 0.4 — Version Control: Git (Mastery)**.*
*Next topic: **Branching, Merging & Rebasing**.*
