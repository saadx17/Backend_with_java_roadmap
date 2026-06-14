# HTTPS & the TLS Handshake

> **Phase 0 — Computer Science Fundamentals → 0.2 Networking Fundamentals**
> Goal: Understand how HTTPS secures HTTP — the TLS handshake, symmetric vs asymmetric encryption, and certificates — so you can reason about secure connections.

---

## 0. The Big Picture

**HTTPS** = **HTTP over TLS**. It's plain HTTP wrapped in a layer of encryption provided by **TLS (Transport Layer Security)** — sitting around OSI Layer 6 (Presentation), between the application (HTTP) and transport (TCP).

```
HTTP    (plaintext)  -->  anyone on the path can read/modify it  ❌
HTTPS = HTTP + TLS   -->  encrypted, authenticated, tamper-proof  ✅
```

> TLS is the successor to the older, now-insecure **SSL** (people still say "SSL" loosely, but modern security is **TLS 1.2 / 1.3**).

---

## 1. What TLS Provides (Three Guarantees)

| Guarantee | Meaning | Protects against |
|-----------|---------|------------------|
| **Confidentiality** | Data is **encrypted** in transit | Eavesdropping (sniffing) |
| **Integrity** | Data can't be **altered** undetected | Tampering / man-in-the-middle modification |
| **Authentication** | You're really talking to the **right server** (via certificates) | Impersonation / spoofing |

> Without HTTPS, anyone between you and the server (Wi-Fi snoopers, ISPs, malicious routers) can **read and modify** your traffic — passwords, tokens, everything in cleartext.

---

## 2. Symmetric vs Asymmetric Encryption (the foundation)

TLS cleverly combines two kinds of cryptography:

| | **Symmetric** | **Asymmetric (public-key)** |
|---|---------------|------------------------------|
| Keys | **One shared** secret key | A **key pair**: public + private |
| Encrypt/decrypt | Same key for both | Public encrypts → private decrypts (and vice-versa for signing) |
| Speed | **Fast** | **Slow** |
| Problem | How to share the key securely? | Solves key sharing, but slow |

### 2.1 The hybrid approach (why TLS is fast *and* secure)
TLS uses **asymmetric** crypto only briefly — to securely **agree on a shared symmetric key** — then uses **fast symmetric** crypto for the actual data:
```
1. Use slow asymmetric crypto to safely establish a shared secret (session key).
2. Use fast symmetric crypto (with that session key) for all the bulk data.
```
> Best of both worlds: asymmetric solves the "how do we share a key over an insecure channel?" problem; symmetric provides fast ongoing encryption.

---

## 3. The TLS Handshake (Step by Step)

After the **TCP 3-way handshake** (recall TCP note) establishes a connection, TLS performs its own handshake to set up encryption **before** any HTTP data flows:

```
TCP handshake (SYN/SYN-ACK/ACK) already done
        |
TLS handshake:
1. Client Hello   -> client sends supported TLS versions, cipher suites, random number
2. Server Hello   <- server picks version/cipher, sends its CERTIFICATE (public key) + random
3. Certificate verification -> client checks the cert is valid & trusted (see §4)
4. Key exchange   -> both derive the same SHARED SESSION KEY (e.g., via Diffie-Hellman)
5. Finished       <- both confirm; switch to symmetric encryption
        |
Encrypted HTTP data flows using the session key
```

| Step | What happens |
|------|--------------|
| **Client Hello** | Client proposes TLS versions + cipher suites + a random value |
| **Server Hello** | Server selects them and sends its **certificate** (containing its public key) |
| **Cert validation** | Client verifies the certificate chain (authentication) |
| **Key exchange** | Both sides derive a shared **session key** (without ever sending it in the clear) |
| **Finished** | Handshake verified; encrypted communication begins |

### 3.1 Handshake cost (performance)
- The handshake adds **round-trips** on top of the TCP handshake → extra latency to start a connection.
- **TLS 1.3** reduced this to **one round-trip** (and 0-RTT for resumption), vs two in TLS 1.2.
- This is why **connection reuse (keep-alive), HTTP/2 multiplexing, and session resumption** matter so much (Phase 13).

---

## 4. Certificates & the Chain of Trust

A **TLS/SSL certificate** binds a domain name to a **public key**, vouched for by a trusted **Certificate Authority (CA)**.

### 4.1 How trust works
```
Browser/OS trusts a set of ROOT CAs (pre-installed trust store)
   Root CA  -> signs ->  Intermediate CA  -> signs ->  Server's certificate (example.com)
                                  (the "certificate chain")
```
- Your browser/OS ships with a list of **trusted root CAs**.
- A server's certificate is signed by a CA in that chain → the client can verify it's legitimate.
- The client checks: **valid signature?**, **not expired?**, **matches the domain?**, **not revoked?**.

### 4.2 What the certificate proves
- That the server **owns the private key** matching the public key in the cert (proven during the handshake).
- That a trusted CA **verified the domain ownership**.
- → You're talking to the **real** `example.com`, not an impostor (authentication).

### 4.3 Common certificate concepts
| Term | Meaning |
|------|---------|
| **CA (Certificate Authority)** | Trusted issuer of certificates (e.g., Let's Encrypt, DigiCert) |
| **Self-signed cert** | Signed by itself, not a CA → browsers warn (OK for local/dev only) |
| **Certificate chain** | Root → intermediate → leaf (server) certs |
| **Let's Encrypt** | Free, automated CA — ubiquitous for HTTPS |
| **Expiry** | Certs have a validity period; must be renewed (automate it!) |
| **SNI** | Server Name Indication — lets one IP serve certs for many domains |

---

## 5. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "HTTPS only encrypts" | It also provides **integrity** and **authentication** |
| "SSL and TLS are the same" | SSL is the old, insecure predecessor; use **TLS 1.2/1.3** |
| "TLS uses only asymmetric crypto" | Hybrid: asymmetric to exchange a key, symmetric for data |
| Ignoring cert expiry | Expired certs break the site — automate renewal |
| Self-signed certs in production | Browsers/clients reject them — use a real CA |
| "HTTPS makes my app fully secure" | It secures **transit only**; app-level security still required (Phase 15) |
| Putting secrets in URLs even over HTTPS | URLs get logged; keep secrets in headers/body |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **TLS termination** (Phase 10, 12): often handled at a **load balancer / reverse proxy / API gateway** (Nginx, ELB), which decrypts and forwards plain HTTP to your app internally.
- **HTTPS everywhere** is mandatory for auth — **JWTs and cookies must travel over HTTPS** (the `Secure` cookie flag — HTTP note; Phase 5.6, 15).
- **Certificates in containers/K8s:** managing certs (cert-manager, Let's Encrypt) is part of deployment (Phase 10).
- **Performance:** TLS handshake cost motivates keep-alive, HTTP/2, and session resumption (Phase 13).
- **mTLS (mutual TLS):** services authenticate *each other* with certs — common in microservice meshes (Phase 12).
- **Security (Phase 15):** HSTS header, strong cipher suites, TLS version policy, and avoiding mixed content.

---

## 7. Quick Self-Check Questions

1. What does HTTPS add to HTTP, and what protocol provides it?
2. What three guarantees does TLS provide?
3. What's the difference between symmetric and asymmetric encryption?
4. Why does TLS use *both* — what does each part do?
5. Walk through the TLS handshake steps.
6. What is a certificate, and how does the chain of trust work?
7. What does a valid certificate prove about the server?
8. Why is TLS handshake cost a performance concern, and what mitigates it?

---

## 8. Key Terms Glossary

- **HTTPS:** HTTP secured with TLS.
- **TLS / SSL:** Transport Layer Security (modern) / its insecure predecessor.
- **Confidentiality / integrity / authentication:** TLS's three guarantees.
- **Symmetric encryption:** one shared key (fast).
- **Asymmetric / public-key encryption:** public + private key pair (slow).
- **Session key:** the shared symmetric key negotiated during the handshake.
- **TLS handshake:** the negotiation that sets up encryption.
- **Certificate:** binds a domain to a public key, signed by a CA.
- **Certificate Authority (CA):** trusted certificate issuer.
- **Certificate chain / root of trust:** root → intermediate → leaf.
- **TLS termination:** decrypting TLS at a proxy/load balancer.
- **mTLS:** mutual TLS where both sides present certificates.

---

*Previous topic: **HTTP Protocol Deep Dive**.*
*Next topic: **Ports, Sockets, Firewalls, Proxies, Load Balancers & CDNs**.*
