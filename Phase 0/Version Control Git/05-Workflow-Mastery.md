# Git Workflow Mastery (.gitignore, Tags, Hooks, Branching Strategies, Conventional Commits, Squashing, Signed Commits)

> **Phase 0 — Computer Science Fundamentals → 0.4 Version Control — Git (Mastery)**
> Goal: Master the professional practices that make Git work at team scale — `.gitignore`, tags, hooks, branching strategies, conventional commits, squashing, and signed commits.

---

## 0. The Big Picture

The earlier notes covered Git *mechanics*. This note covers the *professional conventions and policies* that teams adopt so Git scales cleanly across many people and releases — the difference between "knowing Git commands" and "using Git like a senior engineer."

---

## 1. .gitignore — Keep Junk Out of the Repo

A `.gitignore` file lists patterns of files Git should **not track**. This keeps build artifacts, secrets, and machine-specific files out of version control.

### 1.1 Example (typical Java/Spring project)
```gitignore
# Build output
target/
build/
*.class
*.jar

# IDE files
.idea/
*.iml
.vscode/
.settings/

# Secrets & local config — NEVER commit these
.env
*.pem
application-local.properties

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
```

### 1.2 Pattern syntax
| Pattern | Matches |
|---------|---------|
| `target/` | A directory named target (anywhere) |
| `*.class` | All `.class` files |
| `/secret.txt` | Only at the repo root |
| `!keep.log` | **Negation** — track this despite a broader ignore |
| `**/temp` | `temp` at any depth |

### 1.3 Crucial rules
- `.gitignore` only affects **untracked** files. If a file is **already committed**, adding it to `.gitignore` won't stop tracking it — you must:
  ```bash
  git rm --cached secret.txt    # stop tracking (keeps the local file)
  ```
- ⚠️ **Secrets that were ever committed remain in history** — `.gitignore` doesn't remove them. Removing secrets from history requires rewriting history (`git filter-repo`/BFG) **and rotating the secret** (assume it's compromised). (Phase 15.)
- Use a global ignore for personal/OS files: `git config --global core.excludesfile ~/.gitignore_global`.

> **Golden rule: never commit secrets.** Use environment variables / secret managers (recall 0.3, Phase 5.2/15).

---

## 2. Tags (Marking Releases)

A **tag** is a fixed, named pointer to a specific commit (recall internals note) — used to mark **releases** (e.g., `v1.2.0`).

### 2.1 Lightweight vs annotated tags
```bash
git tag v1.0.0                          # LIGHTWEIGHT: just a name -> commit
git tag -a v1.0.0 -m "Release 1.0.0"    # ANNOTATED: a full object with message/author/date
```
| | Lightweight | Annotated |
|---|-------------|-----------|
| Stores | Just a pointer | A Git object (tagger, date, message, optional signature) |
| Use for | Temporary/private markers | **Releases** (recommended) |

### 2.2 Working with tags
```bash
git tag                       # list tags
git show v1.0.0               # show what a tag points to
git push origin v1.0.0        # push a tag (tags aren't pushed by default!)
git push origin --tags        # push all tags
git checkout v1.0.0           # check out the code at that release
git tag -d v1.0.0             # delete a tag locally
```
> Tags + **Semantic Versioning** (`MAJOR.MINOR.PATCH`, e.g., `v2.1.3`) are the standard for releases — and often trigger release pipelines (Phase 10.3).

---

## 3. Git Hooks (Automation on Git Events)

**Hooks** are scripts Git runs automatically at certain points (commit, push, etc.) — for automating checks like linting, formatting, and tests.

### 3.1 Common hooks
| Hook | Runs | Use |
|------|------|-----|
| `pre-commit` | Before a commit is created | Lint, format, run quick tests, block secrets |
| `commit-msg` | After the message is entered | Enforce commit message format (§5) |
| `pre-push` | Before pushing | Run tests/build to avoid breaking the remote |
| `post-merge` / `post-checkout` | After merge/checkout | e.g., reinstall dependencies |

- Hooks live in `.git/hooks/` (local, not committed by default).
- Tools like **Husky**, **pre-commit**, or **Lefthook** manage *shared* hooks across a team (committed config). In Java, the Maven/Gradle build can enforce similar checks.

```bash
# .git/hooks/pre-commit (example, must be executable)
#!/bin/bash
mvn -q spotless:check || { echo "Formatting failed"; exit 1; }
```
> A non-zero exit from a hook **aborts** the operation — great for "don't let bad code in."

---

## 4. Branching Strategies

A **branching strategy** is the team's agreed convention for how branches are organized and integrated. The three common ones:

### 4.1 Git Flow (structured, release-heavy)
```
main      -> production-ready releases (tagged)
develop   -> integration branch for the next release
feature/* -> branched from develop, merged back
release/* -> prep a release (stabilize, version bump)
hotfix/*  -> urgent prod fixes, branched from main
```
- **Pros:** clear structure for scheduled releases, parallel work.
- **Cons:** complex, many long-lived branches; heavy for continuous delivery.
- **Best for:** versioned products with formal release cycles.

### 4.2 GitHub Flow (simple, continuous)
```
main  -> always deployable
feature branches -> branch from main, PR, merge back, deploy
```
- **Pros:** simple; one main branch + short-lived feature branches; deploy continuously.
- **Cons:** less structure for multiple parallel release versions.
- **Best for:** web apps / SaaS with continuous deployment (most common today).

### 4.3 Trunk-Based Development (high-velocity CI/CD)
```
main (trunk) -> everyone commits small changes frequently (often behind feature flags)
very short-lived branches (hours/a day), merged fast
```
- **Pros:** minimal merge conflicts, fast integration, ideal for CI/CD at scale.
- **Cons:** requires strong test automation, feature flags, discipline.
- **Best for:** mature teams practicing continuous integration.

### 4.4 Choosing
| Strategy | Complexity | Best for |
|----------|-----------|----------|
| Git Flow | High | Scheduled, versioned releases |
| GitHub Flow | Low | Continuous deployment web apps |
| Trunk-based | Low (high discipline) | High-velocity CI/CD teams |

> **Common modern default:** **GitHub Flow** (or trunk-based) — short-lived branches + PRs + CI + continuous deploy. (Ties to Phase 10.3.)

---

## 5. Conventional Commits (Standardized Messages)

**Conventional Commits** is a spec for writing **structured commit messages**, enabling automated changelogs and semantic versioning.

### 5.1 Format
```
<type>(<optional scope>): <description>

[optional body]

[optional footer]
```
```
feat(auth): add JWT refresh token endpoint
fix(orders): prevent negative quantity on checkout
docs(readme): update setup instructions
refactor(user): extract validation into a service
```

### 5.2 Common types
| Type | Meaning |
|------|---------|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `style` | Formatting (no logic change) |
| `refactor` | Code change that's neither feat nor fix |
| `test` | Adding/fixing tests |
| `chore` | Build/tooling/maintenance |
| `perf` | Performance improvement |

- `feat!:` or a `BREAKING CHANGE:` footer signals a breaking change → a **major** version bump.
- Enables tools to **auto-generate changelogs** and **auto-bump semantic versions**.

### 5.3 Good commit message practices (beyond the format)
- **Imperative mood:** "add feature," not "added feature" (matches Git's own messages).
- **Subject line ≤ ~50 chars**, blank line, then a body explaining **why** (not just what).
- One logical change per commit.

---

## 6. Squashing Commits (Clean History)

**Squashing** combines multiple commits into one — turning a messy work-in-progress series ("wip", "fix typo", "actually fix it") into a single clean commit before sharing.

### 6.1 Interactive rebase to squash
```bash
git rebase -i HEAD~4         # edit the last 4 commits
```
In the editor, change `pick` to `squash` (or `s`) / `fixup` (`f`) for commits to fold into the one above:
```
pick   abc123 Add login endpoint
squash def456 fix typo
squash ghi789 address review comments
fixup  jkl012 oops
```
Result: one tidy commit. (`squash` keeps messages to edit; `fixup` discards them.)

### 6.2 Squash-and-merge (on PRs)
Most platforms offer **"Squash and merge"** for PRs — collapsing all branch commits into **one commit on `main`** (recall the PR note). This keeps `main`'s history clean: one commit per feature.

> ⚠️ Squashing rewrites history (new hashes) — same **Golden Rule**: only squash **local/unpushed** commits, or use the platform's squash-merge for the PR itself.

---

## 7. Signed Commits (Verifying Authorship)

**Signed commits** use cryptographic signatures (GPG/SSH keys) to **verify who actually made a commit** — preventing impersonation (anyone can set `user.name`/`user.email` to yours!).

```bash
git commit -S -m "feat: add payment"   # sign this commit
git config --global commit.gpgsign true  # sign all commits by default
git log --show-signature                  # view signatures
```
- Platforms show a **"Verified"** badge for validly signed commits.
- Uses your **GPG or SSH key** (recall asymmetric crypto from the TLS note; SSH keys from 0.3).
- ⚠️ Without signing, commit author fields are **unverified metadata** — trivially spoofable.

> Many organizations **require signed commits** on protected branches for supply-chain security (Phase 15) — it proves a commit genuinely came from you.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Adding a committed file to `.gitignore` expecting it to untrack | `git rm --cached`; ignore only affects untracked files |
| Committing secrets (they persist in history) | Never commit them; if leaked, rotate + scrub history |
| Forgetting to push tags | `git push origin --tags` |
| Lightweight tags for releases | Use **annotated** tags |
| Inconsistent commit messages | Adopt Conventional Commits |
| Squashing already-pushed shared commits | Only squash local work / use PR squash-merge |
| Assuming commit author is trustworthy | Use **signed commits** |
| Over-engineering with Git Flow for a simple app | Use GitHub Flow / trunk-based |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **`.gitignore`** keeps `target/`, `.env`, and IDE files out of every Spring repo (Phase 1, 3, 15).
- **Tags + SemVer** drive release pipelines and artifact versions (Phase 3 Maven versions, Phase 10.3 CD).
- **Hooks / CI checks** enforce formatting, tests, and "no secrets" before code lands (Phase 6, 10.3).
- **Branching strategy + PRs** define the team's daily workflow and release cadence (Phase 10).
- **Conventional Commits** power automated changelogs and semantic-release in CI/CD.
- **Signed commits** are a software supply-chain security control (Phase 15).
- **Squash merges** keep `main` history readable for debugging and changelogs (recall bisect/blame, history note).

---

## 10. Quick Self-Check Questions

1. What does `.gitignore` do, and why won't it untrack an already-committed file?
2. Why is committing a secret dangerous even after deleting the file?
3. What's the difference between a lightweight and an annotated tag? Which for releases?
4. What are Git hooks, and give two examples (pre-commit, pre-push)?
5. Compare Git Flow, GitHub Flow, and Trunk-Based Development. When use each?
6. What is the Conventional Commits format, and what does it enable?
7. How do you squash commits, and what's the Golden Rule when doing so?
8. What problem do signed commits solve, and how?

---

## 11. Key Terms Glossary

- **`.gitignore`:** patterns of files Git shouldn't track.
- **`git rm --cached`:** stop tracking a file without deleting it.
- **Tag (lightweight/annotated):** fixed pointer to a commit / full tag object.
- **Semantic Versioning (SemVer):** `MAJOR.MINOR.PATCH` versioning.
- **Git hook:** script run automatically on a Git event.
- **Branching strategy:** team convention for branch organization.
- **Git Flow / GitHub Flow / Trunk-Based:** the three common strategies.
- **Conventional Commits:** structured commit message spec (`type(scope): desc`).
- **Squashing:** combining multiple commits into one.
- **`git rebase -i`:** interactive rebase (squash/reorder/edit).
- **Squash-and-merge:** collapse a PR's commits into one on merge.
- **Signed commit:** cryptographically verified authorship (GPG/SSH).

---

*Previous topic: **History, Recovery & Inspection**.*
*This completes **Section 0.4 — Version Control: Git (Mastery)** and **all of Phase 0 — CS Fundamentals**.*
*Next: **Phase 1 — Java Language Mastery** (continue with 1.6 Collections, or 1.1 Dev Environment Setup).*
