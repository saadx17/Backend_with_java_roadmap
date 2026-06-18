# Data Protection

> **Phase 15 — Security → 15.3 Data Protection**
> Goal: Protect sensitive data — encryption at rest and in transit, PII handling and masking, GDPR/privacy obligations, and secret management.

---

## 0. The Big Picture

Beyond keeping attackers out (15.1) and controlling access (15.2), you must protect the **data itself** — so that even if a layer is breached, sensitive information stays safe, and you meet legal/privacy obligations. This is OWASP **A02 (Cryptographic Failures)** territory. The core levers: **encrypt in transit, encrypt at rest, minimize/mask PII, and manage secrets properly.**

```
   Data in transit ──TLS──►   |   Data at rest ──encryption──►   |   Secrets ──Vault/KMS──►
   PII ──minimize · mask · retain-only-as-needed · honor user rights (GDPR)──►
```

> Builds on networking/TLS (Phase 0.2), security fundamentals (15.1 — A02), authN/authZ (15.2 — protecting credentials/keys), config/secrets (12.5 — Vault), and logging (9.1 — don't log secrets/PII). Ties to DB (Phase 4) and cloud (16.7 — KMS/managed encryption).

---

## 1. Encryption in Transit ⭐

Protect data moving over the network from eavesdropping/tampering (recall HTTPS — Phase 0.2).
| Practice | Note |
|----------|------|
| **TLS (HTTPS) everywhere** ⭐ | Encrypt all client↔server **and** service↔service traffic (12.2) |
| **Modern TLS (1.2+/1.3)** | Disable old SSL/TLS versions & weak ciphers |
| **HSTS header** | Force HTTPS; prevent downgrade (15.1 §5.3) |
| **Valid certificates** | Use a real CA (Let's Encrypt); automate renewal; cert pinning where apt |
| **mTLS (mutual TLS)** | Both sides authenticate — common in service meshes / zero-trust (12.2/12.4) |
| **Encrypt internal traffic too** | Don't assume the internal network is safe (zero trust — 12.4) |
> ⭐ **TLS is non-negotiable** — never send credentials/tokens/PII over plain HTTP. In Kubernetes, TLS often terminates at the **Ingress/Gateway** (10.2/12.4); **mTLS** (often via a service mesh like Istio) secures internal service-to-service traffic in a zero-trust model.

---

## 2. Encryption at Rest ⭐

Protect stored data so a stolen disk/backup/DB dump is useless without keys.
| Layer | How |
|-------|-----|
| **Disk/volume encryption** | Cloud-managed (EBS/RDS encryption — 16.7), full-disk |
| **Database TDE** (Transparent Data Encryption) | DB encrypts files transparently (Phase 4) |
| **Column/field-level encryption** ⭐ | Encrypt specific sensitive columns (SSN, card) in the app before storing |
| **Backups encrypted** | Don't forget backups & snapshots |
| **Key management (KMS/Vault)** | Keys stored/rotated separately from data (§4) |
```java
// App-level field encryption (e.g., JPA AttributeConverter) for highly sensitive columns
@Convert(converter = EncryptedStringConverter.class)
private String taxId;     // encrypted before INSERT, decrypted on read (key from Vault/KMS — §4)
```
| Decision | Guidance |
|----------|----------|
| Disk/DB encryption | Baseline — enable it (cheap, broad) |
| Field-level encryption | For the most sensitive fields (defense in depth; protects even from DB admins) |
| Hash vs encrypt | **Passwords → hash** (15.2, one-way); **recoverable PII → encrypt** (two-way) |
> ⭐ Use **disk/DB encryption as a baseline** + **field-level encryption** for the crown jewels. ⚠️ **Encryption is only as strong as key management** (§4) — encrypting data with a key sitting next to it in the same config is theater. Don't confuse **hashing** (one-way, for passwords) with **encryption** (reversible, for data you must read back).

---

## 3. PII Handling & Masking

**PII** (Personally Identifiable Information) — names, emails, phone, addresses, government ids, financial data, health data. Handle it with extra care (legal + ethical).
| Practice | Note |
|----------|------|
| **Data minimization** ⭐ | Collect/keep only what you truly need (less PII = less risk) |
| **Masking / redaction** | Show only partial data (`****1234`, `a***@x.com`) in UIs/logs/responses |
| **Tokenization** | Replace PII with a token; real data in a secure vault (common for card data/PCI) |
| **Pseudonymization / anonymization** | Replace identifiers / strip them (for analytics, non-prod data) |
| **Don't log PII** ⚠️ | Scrub before logging (9.1 §6) and before sending to APM/Sentry (9.3) |
| **Mask in non-prod** | Never use real PII in dev/test DBs — mask/synthesize |
| **Access controls + audit** | Restrict who can see PII (15.2); log access (compliance) |
```
   In logs/responses:  "card": "**** **** **** 1234"   (mask)   NOT the full PAN
```
> ⭐ **Data minimization is the cheapest, strongest control** — data you don't store can't leak. **Mask everywhere it's displayed/logged**, **tokenize** the most regulated data (card/PCI), and **never** put real PII in non-prod environments or logs.

---

## 4. Secret Management ⭐

Secrets = passwords, API keys, tokens, encryption keys, certificates. Mishandling them is a top cause of breaches (and an OWASP/CWE classic).
| Practice | Note |
|----------|------|
| ☠️ **Never commit secrets to Git** | `.gitignore` (Phase 0.4); scan repos for leaked secrets (gitleaks/trufflehog) |
| ☠️ **Never bake secrets into images/code** | Inject at runtime (12.5, Phase 10.1) |
| **Use a secret manager** ⭐ | **HashiCorp Vault** (12.5), cloud **KMS/Secrets Manager** (AWS — 16.7) |
| **Dynamic, short-lived secrets** | Vault issues per-use, auto-expiring DB creds (12.5 §3) |
| **Rotate regularly** | Rotate keys/passwords; auto-rotation where possible |
| **Encrypt at rest + tight access** | K8s Secrets are base64, **not** encrypted (10.2 §2.6) → enable encryption-at-rest + RBAC, or use Vault/External Secrets |
| **Separate keys from data** | KMS holds keys; envelope encryption (data key encrypted by a master key) |
| **Audit secret access** | Vault logs every access (12.5) |
> ⭐ **Secrets belong in a secret manager (Vault/KMS), injected at runtime, rotated, and audited** — never in Git/images/logs (15.1). **Envelope encryption** (data encrypted with a data key, which is encrypted by a KMS master key) is the standard pattern for at-rest key management (§2). This is the security side of config management (12.5).

---

## 5. GDPR & Privacy Obligations

**GDPR** (EU) and similar laws (CCPA, etc.) impose legal obligations on handling personal data. Backend engineers must support them technically.
| GDPR principle / right | Technical implication |
|------------------------|------------------------|
| **Lawful basis & consent** | Track consent; collect only with a basis |
| **Data minimization & purpose limitation** | Store only needed data, for stated purposes (§3) |
| **Right to access (DSAR)** | Be able to export a user's data |
| **Right to erasure ("right to be forgotten")** ⭐ | Be able to **delete/anonymize** a user's data on request (hard with backups/event logs — Phase 11.4!) |
| **Right to rectification** | Allow correction |
| **Data portability** | Export in a machine-readable format |
| **Breach notification** | Detect & report breaches within deadlines (logging/monitoring — 9.2) |
| **Privacy by design & default** | Build privacy in from the start |
| **Data residency** | Store data in required regions (cloud regions — 16.7) |
> ⭐ **Right to erasure** is technically tricky: deleting a user means scrubbing them from the DB, **backups**, **caches** (Redis), **search indexes** (Elasticsearch — 16.4), **logs**, and **event stores** (Event Sourcing — 11.4, where events are immutable → use crypto-shredding: delete the user's encryption key so their encrypted data becomes unreadable). Design for these rights up front. ⚠️ This is a **legal** matter — engineers implement, but work with compliance/legal.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Plain HTTP for any sensitive traffic | TLS everywhere (incl. internal) (§1) |
| Encryption key stored next to the data | KMS/Vault; envelope encryption (§2/§4) |
| Confusing hashing and encryption | Hash passwords (one-way); encrypt recoverable PII (§2) |
| Logging PII/secrets | Mask/scrub before logging (§3/§4, 9.1) |
| Real PII in dev/test | Mask/anonymize non-prod data (§3) |
| Secrets in Git/images | Secret manager + runtime injection; secret scanning (§4) |
| K8s Secrets assumed encrypted | base64 ≠ encryption; encryption-at-rest + Vault (§4, 10.2) |
| No key rotation | Rotate regularly (§4) |
| Ignoring right-to-erasure across all stores | Plan deletion incl. backups/logs/events (crypto-shredding) (§5) |
| Treating GDPR as legal-only | Engineers must build the technical capabilities (§5) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **TLS** ↔ networking (0.2), terminated at Ingress/Gateway (10.2/12.4); **mTLS** for zero-trust (12.2).
- **Encryption at rest** ↔ DB (Phase 4), cloud KMS (16.7); field-level via JPA converters (5.4).
- **Don't log secrets/PII** ↔ logging (9.1), APM/Sentry scrubbing (9.3).
- **Secret management** ↔ Vault/config (12.5), K8s Secrets (10.2), API keys/tokens (15.2).
- **PII/GDPR** affects data model (Phase 4), caches (4.8), search (16.4), event stores (11.4 — crypto-shredding).
- OWASP **A02** (15.1); supports **A09** breach detection (9.2).
- Required across **Projects 5/7** (handling user/customer data).

---

## 8. Quick Self-Check Questions

1. What does encryption in transit protect, and what is mTLS?
2. Why encrypt at rest, and what are disk/DB/TDE vs field-level encryption?
3. When do you hash vs encrypt?
4. What is PII, and what are minimization, masking, and tokenization?
5. Why is data minimization the strongest control?
6. Where must you never store secrets, and where should they go?
7. What is envelope encryption, and why separate keys from data?
8. Why are K8s Secrets not secure by default, and how do you fix that?
9. What technical capabilities does GDPR require (access, erasure, portability)?
10. Why is the right to erasure technically hard, and what is crypto-shredding?

---

## 9. Key Terms Glossary

- **Encryption in transit / at rest:** protecting moving / stored data.
- **TLS / mTLS / HSTS:** transport encryption / mutual auth / force-HTTPS.
- **TDE / field-level encryption:** DB transparent / app column encryption.
- **Hashing vs encryption:** one-way (passwords) vs reversible (data).
- **PII:** personally identifiable information.
- **Data minimization / masking / tokenization / pseudonymization:** PII-reduction techniques.
- **Secret management / Vault / KMS:** handling secrets/keys securely.
- **Envelope encryption / data key / master key:** key-wrapping pattern.
- **Key rotation:** periodically replacing keys.
- **GDPR / DSAR / right to erasure / crypto-shredding:** privacy law / data request / deletion / key-deletion to render data unreadable.
- **Privacy by design:** building in privacy from the start.

---

*This is the note for **Section 15.3 — Data Protection**.*
*Previous section in roadmap: **15.2 AuthN/AuthZ Best Practices**.*
*Next section in roadmap: **15.4 Dependency Security**.*
