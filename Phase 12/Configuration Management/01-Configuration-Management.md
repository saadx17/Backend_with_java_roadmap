# Configuration Management (Spring Cloud Config, Vault, K8s)

> **Phase 12 — Microservices → 12.5 Configuration Management**
> Goal: Manage configuration across many services & environments — Spring Cloud Config (centralized, Git-backed, refresh), secret management with HashiCorp Vault, and Kubernetes ConfigMaps/Secrets.

---

## 0. The Big Picture

One service is easy to configure (`application.yml` — Phase 5.2). But **dozens of services × multiple environments** (dev/staging/prod) creates a config-sprawl problem: where does config live, how do you change it without redeploying, and how do you store **secrets** safely? Configuration management centralizes and externalizes config (the **12-factor "config in the environment"** principle).

```
   Centralized config (Git/Vault/ConfigMaps)
            │ services fetch config at startup (and refresh)
            ▼
   Order Svc   Payment Svc   Inventory Svc   ← each gets env-specific values, secrets injected
```

> Builds on Spring externalized config & profiles (Phase 5.1/5.2), Kubernetes ConfigMaps/Secrets (10.2), and security/secrets (Phase 15.3). It's a microservices concern (12.1) — services must stay **stateless** (Phase 13.3) with config & secrets supplied externally.

---

## 1. The Problem & Principles

| Challenge | Goal |
|-----------|------|
| Config duplicated across services | Centralize / standardize |
| Different values per environment | Profiles/overrides (dev/staging/prod) |
| Changing config requires redeploy | **Dynamic refresh** without rebuild |
| Secrets in code/Git ☠️ | **Never** — use a secret manager (§3) |
| Auditing/versioning config | Git-backed history (§2) |

> ⭐ **12-factor app principles** here: strict separation of **config from code**; config comes from the **environment**; the **same artifact** runs in every env (build once, deploy many — Phase 10.3) with different config. ⚠️ **Never commit secrets to Git** (`.gitignore` — Phase 0.4; info disclosure — Phase 15.3).

---

## 2. Spring Cloud Config

A **centralized config server** that serves configuration to all services, typically backed by a **Git repository** (so config is versioned, auditable, reviewable — Phase 0.4).
```
   Git repo (config)                Config Server                 Services (clients)
   order-service-prod.yml  ──►  [ Spring Cloud Config ]  ◄── fetch on startup ── Order Service
   payment-service.yml                                    ◄── /actuator/refresh ── Payment Service
```
**Server:**
```java
@SpringBootApplication
@EnableConfigServer            // this app IS the config server
public class ConfigServerApp { ... }
```
```yaml
spring.cloud.config.server.git.uri: https://github.com/org/config-repo
```
**Client (each service):**
```yaml
spring.config.import: optional:configserver:http://config-server:8888
spring.application.name: order-service     # → fetches order-service.yml + order-service-<profile>.yml
```
| Concept | Note |
|---------|------|
| **Git backend** | Versioned, auditable, PR-reviewed config (or Vault/filesystem backends) |
| **Naming** | `{application}-{profile}.yml` (profiles — Phase 5.1) |
| **Refresh** | `@RefreshScope` beans + `POST /actuator/refresh` reload config **without restart** ⭐ |
| **Spring Cloud Bus** | Broadcast a refresh to *all* instances at once (via Kafka/RabbitMQ — Phase 11) |
| **Encryption** | Server can encrypt/decrypt property values (`{cipher}...`) |
```java
@RefreshScope                  // beans re-created on /actuator/refresh → pick up new config
@Component
class FeatureFlags { @Value("${feature.x.enabled}") boolean xEnabled; }
```
> ⭐ **Dynamic refresh** is the headline feature: change a value in Git → `/actuator/refresh` (or Spring Cloud Bus to all instances) → services pick it up **without redeploying**. Great for feature flags & tuning. ⚠️ The config server is a dependency — make it HA; clients should tolerate it being down (`optional:`, cached/local fallback).

---

## 3. HashiCorp Vault (Secret Management) ⭐

Config servers and ConfigMaps aren't designed for **secrets**. **Vault** is a dedicated secret manager: encrypted storage, fine-grained access policies, audit logs, **dynamic secrets**, and **leasing/rotation**.
| Vault feature | Why it matters |
|---------------|----------------|
| **Encrypted secret storage** | Secrets encrypted at rest (Phase 15.3) |
| **Dynamic secrets** ⭐ | Generates **short-lived, on-demand** credentials (e.g., a DB user that auto-expires) — no long-lived passwords |
| **Leasing & rotation** | Secrets have a TTL and are rotated automatically |
| **Access policies / auth** | Fine-grained: which service can read which secret (RBAC) |
| **Audit log** | Every secret access is logged (compliance — Phase 15.3) |
| **Transit / encryption-as-a-service** | Encrypt data without handling keys |
```yaml
# Spring Cloud Vault — fetch secrets at startup
spring.cloud.vault:
  uri: https://vault:8200
  authentication: KUBERNETES        # authenticate using the service's K8s identity
  kv: { backend: secret }
# → spring.datasource.password is injected from Vault, never in Git/image
```
> ⭐ **Dynamic secrets** are Vault's superpower: instead of a static DB password sitting in config forever, Vault issues a **unique, short-lived** credential per service/session and revokes it on expiry — drastically reducing the blast radius of a leak (Phase 15.3). This is the gold standard for secret management.

---

## 4. Kubernetes ConfigMaps & Secrets

If you run on Kubernetes (10.2), it provides native config & secret objects — often enough without a separate config server.
| Resource | For | Injected as |
|----------|-----|-------------|
| **ConfigMap** | Non-secret config | Env vars or mounted files |
| **Secret** | Sensitive values | Env vars or mounted files (⚠️ base64, not encrypted by default — §4.1) |
```yaml
# Deployment (10.2) consuming config + secrets
envFrom:
  - configMapRef: { name: order-config }
  - secretRef:    { name: order-secrets }
```
Spring Boot reads these as standard environment-variable property sources (Phase 5.2). The **Spring Cloud Kubernetes** project can even map ConfigMaps/Secrets to property sources and trigger refresh on change.

### 4.1 ⚠️ K8s Secrets are NOT encrypted by default
```
   K8s Secret value = base64(plaintext)   ← encoding, NOT encryption! (recall 10.2 §2.6)
```
- Enable **encryption-at-rest** for etcd, restrict **RBAC** to Secrets, and/or use an **external secret store**:
  - **External Secrets Operator** / **Vault Agent / CSI driver** → sync real secrets from Vault (§3) into K8s.
  - Cloud secret managers (AWS Secrets Manager — Phase 16.7).
> ⚠️ **Don't treat base64 as security.** For real secret hygiene, integrate **Vault** (§3) or a cloud secret manager + encryption-at-rest + tight RBAC (Phase 15.3).

---

## 5. Choosing an Approach

| Situation | Use |
|-----------|-----|
| Non-K8s, many services, want Git-versioned config + refresh | **Spring Cloud Config** (§2) |
| Running on Kubernetes, simple config | **ConfigMaps/Secrets** (§4) — often enough |
| Any environment, serious secret management | **Vault** (§3) (+ dynamic secrets/rotation) |
| Cloud-native | Cloud config/secret services (Parameter Store, Secrets Manager — 16.7) |
> ⭐ Common modern stack: **ConfigMaps** for plain config + **Vault (or External Secrets)** for secrets, on **Kubernetes**. Spring Cloud Config is great off-K8s or when you want Git-backed config with dynamic refresh + Spring Cloud Bus.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Secrets committed to Git / baked into images | Vault / K8s Secrets / `.gitignore` (§1/§3, Phase 15.3) |
| Treating K8s Secrets as encrypted | base64 ≠ encryption; encryption-at-rest + Vault (§4.1) |
| Config changes require redeploy | Dynamic refresh (`@RefreshScope` + bus) (§2) |
| Per-env rebuilds of the artifact | Same artifact + external config (12-factor) (§1, Phase 10.3) |
| Config server as a SPOF | Make it HA; clients cache/`optional:` fallback (§2) |
| Long-lived static DB passwords | Vault dynamic secrets + rotation (§3) |
| No audit of secret access | Vault audit log (§3, Phase 15.3) |
| Stateful services holding config in memory permanently | Keep stateless; refresh externally (Phase 13.3) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- Extends Spring **externalized config & profiles** (Phase 5.1/5.2) to many services.
- **K8s ConfigMaps/Secrets** (10.2) inject config; **Deployments** consume them.
- **Vault** = secret management → ties to **data protection/encryption/secrets** (Phase 15.3) and **dynamic DB creds** (Phase 4).
- **Spring Cloud Bus** broadcasts refresh via **Kafka/RabbitMQ** (Phase 11).
- Keeps services **stateless** for horizontal scaling (Phase 13.3) and clean **CI/CD** (build once — Phase 10.3).
- **Security**: never leak secrets (Phase 15.3); restrict access (RBAC).
- Used across **Project 7 (Microservices E-Commerce)**.

---

## 8. Quick Self-Check Questions

1. What config problems arise at microservices scale, and what 12-factor principle addresses them?
2. What is Spring Cloud Config, and why is a Git backend valuable?
3. How does dynamic refresh work (`@RefreshScope`, `/actuator/refresh`, Spring Cloud Bus)?
4. What does Vault add over plain config, and what are dynamic secrets?
5. Why are dynamic, short-lived secrets safer than static passwords?
6. How do K8s ConfigMaps and Secrets inject config into a service?
7. Why are K8s Secrets not actually secure by default, and how do you fix that?
8. How would you choose between Spring Cloud Config, K8s ConfigMaps/Secrets, and Vault?
9. Why must the config server be highly available, and how do clients tolerate its absence?
10. Why does config management depend on services being stateless?

---

## 9. Key Terms Glossary

- **Configuration management:** centralized, externalized config across services/envs.
- **12-factor config:** config separated from code, supplied by the environment.
- **Spring Cloud Config (Server/Client):** centralized Git-backed config service.
- **`@RefreshScope` / `/actuator/refresh` / Spring Cloud Bus:** dynamic reload / endpoint / broadcast.
- **HashiCorp Vault:** dedicated secret manager.
- **Dynamic secrets / leasing / rotation:** on-demand short-lived credentials.
- **ConfigMap / Secret (K8s):** non-secret config / sensitive config objects.
- **Encryption-at-rest / External Secrets Operator:** protect Secrets / sync from Vault.
- **Stateless service:** holds no local state; config/secrets external.

---

*This is the note for **Section 12.5 — Configuration Management**.*
*Previous section in roadmap: **12.4 API Gateway**.*
*Next section in roadmap: **12.6 Distributed Patterns**.*
