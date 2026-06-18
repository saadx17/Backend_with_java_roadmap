# CI/CD (Continuous Integration & Delivery/Deployment)

> **Phase 10 — Containerization & Deployment → 10.3 CI/CD**
> Goal: Understand CI/CD — pipeline stages, **GitHub Actions** (workflows/jobs/steps/secrets/matrix), **GitLab CI** and **Jenkins**, code quality with **SonarQube**, and artifact repositories.

---

## 0. The Big Picture

**CI/CD** automates the path from a Git commit (Phase 0.4) to running software — building, testing, scanning, packaging, and deploying — so releases are **fast, repeatable, and safe** instead of manual and error-prone.

```
   git push ──► CI ─────────────────────────► CD ──────────────────►
   ┌────────────────────────────────────┐   ┌──────────────────────┐
   │ build → test → scan → package image │   │ deploy to staging →   │
   │ (Maven/Gradle · JUnit · Sonar ·     │   │  (auto) → prod        │
   │  Trivy · Docker/Jib)                │   │  (Rolling/Canary 10.2)│
   └────────────────────────────────────┘   └──────────────────────┘
```

| Term | Meaning |
|------|---------|
| **CI (Continuous Integration)** | Every commit is automatically **built + tested** (merge small/often → catch breakage early) |
| **Continuous Delivery** | Every passing build is **deployable**; release to prod is a **manual** button |
| **Continuous Deployment** | Every passing build **auto-deploys** to prod (no manual gate) |

> CI/CD ties together everything: builds (Phase 3 Maven/Gradle), tests (Phase 6), Docker images (10.1), K8s deploys (10.2), DB migrations as Jobs (Phase 8), security scans (Phase 15.4), and contract checks (OpenAPI — 7.2). Built on Git (Phase 0.4).

---

## 1. Pipeline Stages (a typical Java backend)

```
1. Checkout      git clone the repo (Phase 0.4)
2. Setup         JDK 21, cache ~/.m2 (Phase 3)
3. Build         mvn -B verify  (compile)
4. Test          unit + integration (JUnit/Testcontainers — Phase 6) → coverage
5. Quality       SonarQube/lint, coverage gate (§5)
6. Security      dependency scan (OWASP/Snyk — 15.4) + image scan (Trivy — 10.1)
7. Package       build Docker image (Jib/Dockerfile — 10.1) → tag
8. Publish       push image to registry; jar to artifact repo (§6)
9. Deploy        kubectl/Helm to staging (10.2) → smoke tests → prod (gated)
```
> ⭐ **Fail fast & cheap:** run quick checks (compile, unit tests, lint) first; expensive ones (integration, image build, deploy) later. A red pipeline **blocks the merge** (branch protection — Phase 0.4). The artifact built **once** is promoted unchanged through environments (build once, deploy many).

---

## 2. GitHub Actions ⭐

GitHub's built-in CI/CD. **Workflows** (YAML in `.github/workflows/`) contain **jobs** (run on **runners**), each with **steps** (actions or shell). Triggered by **events** (push, PR, schedule).

```yaml
name: CI
on:                                    # EVENTS that trigger this workflow
  push: { branches: [main] }
  pull_request: { branches: [main] }

jobs:
  build-test:                          # a JOB (runs on a fresh runner)
    runs-on: ubuntu-latest             # the RUNNER
    steps:
      - uses: actions/checkout@v4      # reusable ACTION (step)
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '21', cache: maven }   # cache ~/.m2 (Phase 3)
      - run: mvn -B verify             # a shell STEP (build + test — Phase 6)
      - uses: actions/upload-artifact@v4
        with: { name: jar, path: target/*.jar }

  docker:
    needs: build-test                  # runs only after build-test passes (dependency)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "${{ secrets.REGISTRY_TOKEN }}" | docker login -u ci --password-stdin   # SECRET
      - run: docker build -t registry/order:${{ github.sha }} . && docker push registry/order:${{ github.sha }}
```

| Concept | Meaning |
|---------|---------|
| **Workflow** | A YAML automation file (`.github/workflows/*.yml`) |
| **Event/trigger** | `push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual) |
| **Job** | A set of steps on one runner; jobs run in parallel unless `needs:` chains them |
| **Step** | A single task: a `run:` shell command or a `uses:` reusable **action** |
| **Runner** | The machine executing a job (GitHub-hosted or self-hosted) |
| **Secret** | Encrypted variable (`${{ secrets.X }}`) — tokens/passwords (Phase 15.3) |
| **Artifact** | Files passed between jobs / kept after a run |
| **Cache** | Persist deps (`~/.m2`) between runs for speed |

### 2.1 Matrix builds ⭐
Run the same job across multiple combinations (JDK versions, OSes) in parallel:
```yaml
strategy:
  matrix:
    java: [17, 21]                     # test on both LTS versions (Phase 1.1)
runs-on: ubuntu-latest
steps:
  - uses: actions/setup-java@v4
    with: { distribution: temurin, java-version: ${{ matrix.java }} }
  - run: mvn -B verify
```
> ⭐ A **matrix** fans out one job into many variants (e.g., JDK 17 + 21) — great for verifying compatibility. ⚠️ Never put secrets in logs/PRs from forks; restrict secret access for untrusted PRs.

---

## 3. GitLab CI & Jenkins (alternatives)

### 3.1 GitLab CI/CD
Defined in **`.gitlab-ci.yml`**; **stages** contain **jobs** run by **runners**; uses `artifacts`, `cache`, `rules`, and built-in environments.
```yaml
stages: [build, test, deploy]
build:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script: mvn -B package
  artifacts: { paths: [target/*.jar] }
```
> Tightly integrated with GitLab repos; built-in container registry & environments. Conceptually mirrors GitHub Actions (stages≈jobs ordering, jobs, runners).

### 3.2 Jenkins
The veteran, self-hosted, **plugin**-rich automation server. Pipelines as code in a **`Jenkinsfile`** (Groovy).
```groovy
pipeline {
  agent any
  stages {
    stage('Build') { steps { sh 'mvn -B package' } }
    stage('Test')  { steps { sh 'mvn -B verify' } }
    stage('Deploy'){ steps { sh 'kubectl apply -f k8s/' } }   // (10.2)
  }
}
```
| | GitHub Actions | GitLab CI | Jenkins |
|--|----------------|-----------|---------|
| Hosting | SaaS (GitHub) | SaaS/self | **Self-hosted** |
| Config | `.github/workflows/*.yml` | `.gitlab-ci.yml` | `Jenkinsfile` (Groovy) |
| Strength | Easy, marketplace actions | All-in-one DevOps | Ultra-flexible, plugins, on-prem |
| Trade-off | GitHub-centric | GitLab-centric | You operate/maintain it |
> ⭐ All three express the **same pipeline concepts** (stages/jobs/steps, runners/agents, secrets, artifacts). Learn the model once; the syntax differs.

---

## 4. (CD) Deployment Automation

- **Push-based:** the pipeline runs `kubectl apply`/`helm upgrade` to the cluster (10.2).
- **Pull-based / GitOps** (Argo CD, Flux): the cluster **watches a Git repo** and reconciles to it — Git is the single source of truth (recall K8s declarative model — 10.2; Git — Phase 0.4).
- **Promotion:** the **same image** (tagged by commit SHA) flows staging → prod (build once, deploy many).
- **Strategies:** Rolling/Blue-Green/Canary (10.2 §9) with smoke tests + auto-rollback on failed health/metrics (9.2).
> ⭐ **GitOps** is the modern CD pattern: commit a manifest change → Argo CD/Flux deploys it; rollback = `git revert`. Pairs perfectly with K8s.

---

## 5. SonarQube (Code Quality Gate)

**SonarQube** does **static analysis** — finding bugs, code smells, security **vulnerabilities/hotspots**, duplication, and tracking **test coverage** — and enforces a **Quality Gate** that can **fail the build**.
```yaml
- run: mvn -B verify sonar:sonar -Dsonar.projectKey=order -Dsonar.host.url=$SONAR_URL
  env: { SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }} }
```
| SonarQube checks | |
|------------------|--|
| **Bugs** | Likely runtime errors |
| **Code smells** | Maintainability issues (ties to Clean Code — Phase 14.3) |
| **Vulnerabilities / security hotspots** | Security flaws (Phase 15.1) |
| **Coverage** | % of code exercised by tests (Phase 6.1) |
| **Duplication** | Copy-pasted code |
| **Quality Gate** | Pass/fail threshold (e.g., "coverage on new code ≥ 80%, 0 new bugs") |
> ⭐ The **Quality Gate** is the enforcement: a PR that drops coverage or adds vulnerabilities **fails CI** and can't merge (branch protection — Phase 0.4). Focus gates on **new code** ("clean as you code"). Alternatives/complements: SpotBugs, PMD, Checkstyle, ArchUnit (Phase 6.7).

---

## 6. Artifact Repositories

Built artifacts (JARs, Docker images) must be **stored & versioned** for distribution and reproducibility.
| Repo | Stores |
|------|--------|
| **Maven/Java artifacts** | **Nexus**, **Artifactory**, GitHub Packages — your jars/libraries (Phase 3) |
| **Container images** | Docker Hub, **AWS ECR**, **GHCR**, GitLab Registry, Harbor (10.1) |
| **Why** | Versioned, immutable, shareable across teams/services; cache for build speed; proxy public repos |
```
mvn deploy        → publishes the jar to Nexus/Artifactory (internal library reuse)
docker push       → publishes the image to a container registry (10.1 → run in K8s 10.2)
```
> ⭐ Tag artifacts **immutably** (version or commit SHA — never overwrite a released tag), so a deployed version is always traceable to a commit (Phase 0.4) and reproducible. Registries also feed **image scanning** (Trivy — 10.1/15.4).

---

## 7. CI/CD Best Practices

| Practice | Why |
|----------|-----|
| **Build once, deploy many** | Same artifact through all envs (no per-env rebuild drift) |
| **Fail fast** (cheap checks first) | Quick feedback; save runner minutes |
| **Cache dependencies** (`~/.m2`) | Faster builds (Phase 3) |
| **Pin action/tool versions** | Reproducibility & supply-chain safety |
| **Secrets in the secret store** (never in YAML/Git) | Security (Phase 15.3); `.gitignore` (Phase 0.4) |
| **Branch protection + required checks** | No merge unless green (Phase 0.4) |
| **Run security & quality scans** | Sonar, OWASP/Snyk, Trivy (Phase 15.4) |
| **Test in a real DB** (Testcontainers) | Trustworthy tests (Phase 6.5) |
| **Smoke test after deploy + auto-rollback** | Catch bad releases fast (9.2/10.2) |
| **Keep pipelines fast** | Slow CI kills the feedback loop |

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Secrets hardcoded in workflow YAML | Use the secret store (§2/§7) |
| Rebuilding the artifact per environment | Build once, promote the same image (§4/§7) |
| Slow pipelines (no caching) | Cache deps; parallelize jobs (§2) |
| No quality/security gate | SonarQube gate + dependency/image scans (§5, 15.4) |
| Tests on H2, not real DB | Testcontainers in CI (Phase 6.5) |
| Mutable image tags (`:latest`) | Immutable SHA/version tags (§6, 10.1) |
| Manual deploys | Automate (kubectl/Helm/GitOps) (§4) |
| No rollback plan | Smoke tests + `rollout undo`/Argo rollback (9.2/10.2) |
| Forks accessing secrets | Restrict secrets for untrusted PRs (§2.1) |
| Confusing delivery vs deployment | Delivery = deployable + manual gate; deployment = auto (§0) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- Runs **Maven/Gradle** builds (Phase 3) and **tests** (Phase 6 — incl. Testcontainers 6.5).
- Builds/scans **Docker images** (10.1 — Jib/Trivy) and deploys to **Kubernetes** (10.2 — kubectl/Helm, strategies).
- Runs **DB migrations as Jobs** (Phase 8) and **OpenAPI contract/diff checks** (7.2).
- **SonarQube** enforces **Clean Code** (Phase 14.3) & catches **security** issues (Phase 15.1).
- **Dependency scanning** (OWASP/Snyk/Dependabot — Phase 15.4) lives here.
- **Secrets** management (Phase 12.5/15.3); **observability** smoke tests (9.2).
- Built on **Git** workflows & branch protection (Phase 0.4).
- The whole point of **Project 6 (Deployment Pipeline)**; used in **Project 7 (Microservices)**.

---

## 10. Quick Self-Check Questions

1. Define CI, Continuous Delivery, and Continuous Deployment — how do they differ?
2. List the typical stages of a Java backend pipeline and why order matters (fail fast).
3. In GitHub Actions, what are workflows, jobs, steps, runners, and how do `needs:` and secrets work?
4. What is a matrix build, and when is it useful?
5. Compare GitHub Actions, GitLab CI, and Jenkins (hosting, config, strengths).
6. What is GitOps, and how does pull-based CD differ from push-based?
7. What does SonarQube analyze, and what does a Quality Gate enforce?
8. Why are artifact repositories needed, and why use immutable tags?
9. What does "build once, deploy many" mean and why does it matter?
10. How do CI/CD, deployment strategies (10.2), and DB migrations (Phase 8) fit together for safe releases?

---

## 11. Key Terms Glossary

- **CI / CD (delivery vs deployment):** auto build+test / deployable-with-gate vs auto-to-prod.
- **Pipeline / stage / job / step:** the automation flow and its units.
- **GitHub Actions / workflow / runner / action:** GitHub CI / YAML file / executor / reusable step.
- **`needs:` / matrix:** job dependency / fan-out across variants.
- **Secret:** encrypted CI variable (tokens/passwords).
- **Artifact / cache:** files passed/kept between runs / persisted deps.
- **GitLab CI (`.gitlab-ci.yml`) / Jenkins (`Jenkinsfile`):** alternative CI systems.
- **GitOps (Argo CD / Flux):** cluster reconciles to a Git repo (pull-based CD).
- **SonarQube / Quality Gate:** static analysis / pass-fail threshold.
- **Artifact repository (Nexus/Artifactory) / container registry (ECR/GHCR):** jar / image stores.
- **Build once, deploy many:** promote one immutable artifact across environments.
- **Smoke test / rollback:** post-deploy sanity check / revert a bad release.

---

*This is the note for **Section 10.3 — CI/CD**.*
*Previous section in roadmap: **10.2 Kubernetes**.*
*This completes **Phase 10 — Containerization & Deployment**.*
*Next: **Phase 11 — Messaging**.*
