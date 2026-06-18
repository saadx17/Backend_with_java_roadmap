# Dependency Security (Supply Chain)

> **Phase 15 — Security → 15.4 Dependency Security**
> Goal: Secure the software supply chain — vulnerability scanning (OWASP Dependency-Check, Snyk, Dependabot, Trivy), keeping dependencies updated, and managing license compliance.

---

## 0. The Big Picture

Modern apps are mostly **other people's code** — a Spring Boot app pulls in hundreds of transitive dependencies (Phase 3). A vulnerability in **any** of them is *your* vulnerability. This is OWASP **A06 (Vulnerable & Outdated Components)** and **A08 (Software & Data Integrity / supply chain)**. **Dependency security** = continuously knowing what you depend on and whether it's safe.

```
   Your code (5%)
   + direct deps (Spring, Jackson, ...) 
   + TRANSITIVE deps (deps of deps...)   ← the huge, invisible majority
   = your real attack surface  →  scan it, patch it, watch it
```

> Builds on build tools (Phase 3 — Maven/Gradle dependency trees), CI/CD (10.3 — scanning gates), Docker (10.1 — image scanning/Trivy), and the OWASP framing (15.1). The infamous **Log4Shell** (Log4j) and **Spring4Shell** incidents made this a first-class concern.

---

## 1. Why It Matters — Known CVEs ⭐

A **CVE** (Common Vulnerabilities and Exposures) is a publicly catalogued vulnerability with an ID and **CVSS** severity score. Once a CVE is public, attackers actively scan the internet for unpatched systems.
| Reality | Implication |
|---------|-------------|
| Transitive deps dominate | You're exposed to code you never chose directly |
| Public CVE = race | Patch before attackers exploit (often within hours/days) |
| **Log4Shell (CVE-2021-44228)** | One logging lib's flaw → mass RCE across the internet (9.1) |
| Old versions accumulate risk | "It works, don't touch it" is a security debt |
> ⭐ **You can't secure what you don't know you have.** The first step is an inventory (an **SBOM** — Software Bill of Materials) and continuous scanning against vulnerability databases (NVD). Most breaches via dependencies are **known, patchable** CVEs that simply weren't updated.

---

## 2. Vulnerability Scanning Tools ⭐

Tools cross-reference your dependency tree (and container images) against CVE databases and flag known-vulnerable versions.
| Tool | Scans | Note |
|------|-------|------|
| **OWASP Dependency-Check** ⭐ | Java/JS/etc. deps vs NVD | Free, OSS; Maven/Gradle plugin; CI-friendly |
| **Snyk** | Deps + containers + IaC + code | Commercial (free tier); great DB, fix suggestions/PRs |
| **GitHub Dependabot** ⭐ | Deps in your repo | Built into GitHub; **alerts + auto-PRs** to bump versions |
| **Trivy** ⭐ | **Container images** + deps + IaC + secrets | Fast, OSS; scan Docker images (10.1) in CI |
| **OWASP / GitHub Advisory DB, NVD** | The data sources | What the tools query |
```xml
<!-- OWASP Dependency-Check Maven plugin (Phase 3) -->
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <configuration><failBuildOnCVSS>7</failBuildOnCVSS></configuration>  <!-- fail on high severity -->
</plugin>
```
```bash
trivy image myapp:1.0            # scan a built Docker image for CVEs (10.1/10.3)
```
> ⭐ Use a **layered set**: **Dependabot** (auto-PRs to update deps) + **Dependency-Check or Snyk** (deep CVE scan in CI) + **Trivy** (scan the final container image — 10.1). Run them in **CI/CD** (10.3) so vulnerable builds **fail the pipeline** (like the SonarQube/quality gate — 10.3). ⚠️ Tune severity thresholds & **triage false positives** (not every flagged CVE is reachable/exploitable in your usage).

---

## 3. Keeping Dependencies Updated ⭐

| Practice | Why |
|----------|-----|
| **Update regularly** ⭐ | Small frequent updates < painful big-bang upgrades |
| **Automated update PRs** (Dependabot/Renovate) | Surfaces updates continuously |
| **Pin/lock versions** | Reproducible builds (Maven BOM / Gradle lockfiles — Phase 3) |
| **Use a BOM** | Spring Boot's dependency BOM keeps versions compatible (Phase 3/5.2) |
| **Test before merging updates** | CI tests (Phase 6) catch breaking changes |
| **Track EOL** | Don't run end-of-life libraries/runtimes (e.g., old Java — patch to LTS, Phase 1.1) |
| **SBOM generation** | CycloneDX/SPDX — inventory for audit & rapid response |
> ⭐ The cheapest way to stay secure is to **stay current** — automated PRs + good test coverage (Phase 6) make updates routine rather than scary. ⚠️ When a critical CVE drops (e.g., Log4Shell), you need to **answer "are we affected?" in minutes** — an SBOM + scanning gives you that. Pin versions for reproducibility, but don't let them rot.

---

## 4. License Compliance ⭐

Open-source licenses impose **legal obligations**. Using a dependency with an incompatible license can create legal/IP risk for your product.
| License type | Examples | Implication |
|--------------|----------|-------------|
| **Permissive** | MIT, Apache 2.0, BSD | ✅ Generally safe for commercial use (attribution) |
| **Weak copyleft** | LGPL, MPL, EPL | Conditions if you modify the library |
| **Strong copyleft** | **GPL, AGPL** ⚠️ | May require **open-sourcing your code** (AGPL even over a network) — often disallowed in proprietary products |
| **Proprietary/commercial** | Various | Require a paid license |
> ⭐ **Scan licenses** of your dependency tree (Snyk, FOSSA, Maven license plugins) and have an **allow/deny policy** (e.g., permit MIT/Apache, block AGPL). ⚠️ **AGPL** is the classic trap — it can obligate you to release source even for a network service. License compliance is a **legal** concern engineers must support with tooling/inventory.

---

## 5. Supply Chain Integrity (briefly)

Beyond known CVEs, the supply chain itself can be attacked (A08):
| Threat | Mitigation |
|--------|------------|
| **Typosquatting / malicious packages** | Verify package names/sources; use trusted registries (Phase 10.3) |
| **Compromised build/dependency** | Pin versions + checksums; verify signatures |
| **Dependency confusion** | Scope internal packages; lock private registries |
| **Tampered images** | Sign/verify images; scan (Trivy); use trusted base images (10.1) |
| **SBOM + provenance (SLSA)** | Know & attest what's in your build |
> ⭐ Use **trusted registries** (Nexus/Artifactory — 10.3), **pin + verify checksums/signatures**, generate an **SBOM**, and scan images (10.1). Supply-chain security is increasingly mandated (executive orders, SLSA framework).

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Ignoring transitive dependencies | Scan the full tree (SBOM) (§1/§2) |
| "It works, don't update" | Update regularly; automate (§3) |
| No scanning in CI | Add Dependency-Check/Snyk/Trivy gates (§2, 10.3) |
| Only scanning app deps, not images | Trivy-scan container images too (§2, 10.1) |
| Drowning in / ignoring all CVE alerts | Triage by severity & reachability (§2) |
| Running EOL Java/libraries | Upgrade to supported LTS (§3, Phase 1.1) |
| Ignoring licenses (AGPL trap) | License scanning + policy (§4) |
| Trusting any package/registry | Trusted registries, pin + verify (§5) |
| No way to answer "are we affected?" | Maintain an SBOM (§1/§3) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- Builds on **Maven/Gradle** dependency management & BOMs (Phase 3, 5.2).
- **CI/CD** (10.3) runs scans as gates (alongside SonarQube — 10.3); **Trivy** scans **Docker images** (10.1).
- OWASP **A06/A08** (15.1); complements code-level security (15.1) and secrets (15.3 — secret scanning).
- **Java LTS** currency (Phase 1.1); reproducible builds support **build-once-deploy-many** (10.3).
- Incident response (Log4Shell) ties to **logging** (9.1) and **monitoring** (9.2).
- Required across all **Projects** (esp. 6 — pipeline includes scanning).

---

## 8. Quick Self-Check Questions

1. Why are dependencies (especially transitive) a major attack surface?
2. What is a CVE/CVSS, and why is a public CVE a race?
3. What was Log4Shell, and what lesson did it teach?
4. Compare OWASP Dependency-Check, Snyk, Dependabot, and Trivy.
5. How do you wire dependency scanning into CI/CD as a gate?
6. Why update dependencies regularly, and how do automation + BOM help?
7. What is an SBOM, and why is it critical for incident response?
8. Compare permissive vs copyleft licenses; what's the AGPL trap?
9. Name three supply-chain threats and their mitigations.
10. How do you triage CVE alerts without alert fatigue?

---

## 9. Key Terms Glossary

- **Dependency / supply-chain security:** securing third-party and transitive code.
- **Transitive dependency:** a dependency of a dependency.
- **CVE / CVSS / NVD:** catalogued vulnerability / severity score / national vuln database.
- **OWASP Dependency-Check / Snyk / Dependabot / Trivy:** dependency & image scanners.
- **SBOM (CycloneDX/SPDX):** software bill of materials (inventory).
- **BOM (Bill of Materials):** managed compatible version set (Maven/Gradle).
- **Permissive / copyleft (GPL/AGPL) licenses:** OSS license categories.
- **Typosquatting / dependency confusion:** malicious-package supply-chain attacks.
- **SLSA / provenance:** supply-chain integrity framework / build attestation.
- **EOL:** end-of-life (unsupported) software.

---

*This is the note for **Section 15.4 — Dependency Security**.*
*Previous section in roadmap: **15.3 Data Protection**.*
*This completes **Phase 15 — Security**.*
*Next: **Phase 16 — Advanced Topics**.*
