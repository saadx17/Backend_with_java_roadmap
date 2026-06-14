# DNS (Domain Name System)

> **Phase 0 — Computer Science Fundamentals → 0.2 Networking Fundamentals**
> Goal: Understand how human-friendly domain names are resolved to IP addresses — the resolution process, record types (A, CNAME, etc.), and TTL/caching.

---

## 0. The Big Picture

**DNS** is the internet's "phone book": it translates human-readable **domain names** (`example.com`) into machine-usable **IP addresses** (`93.184.216.34`). Without DNS, you'd have to memorize IP addresses for every website.

```
"example.com"  --(DNS resolution)-->  93.184.216.34  --(connect)-->  server
```

DNS operates at the **Application layer (OSI L7)** and is a prerequisite for almost every network connection.

---

## 1. Why DNS Exists

- Humans remember **names** (`google.com`); computers route via **IP addresses**.
- IPs can **change** (servers move, scale, fail over) — a stable name can point to changing IPs.
- One name can map to **many** servers (load balancing, CDNs).

> DNS adds a layer of **indirection** between names and addresses — the same flexibility benefit you see throughout computing.

---

## 2. The DNS Hierarchy

DNS is a **distributed, hierarchical** database — no single server holds everything. Names are read **right to left**:
```
www.example.com.
 |    |     |   |
 |    |     |   +-- Root (the implicit trailing dot)
 |    |     +------ TLD (Top-Level Domain): .com
 |    +------------ Second-level domain: example
 +----------------- Subdomain/host: www
```

| Level | Managed by | Example |
|-------|-----------|---------|
| **Root** | Root name servers (13 logical clusters) | `.` |
| **TLD** | TLD registries | `.com`, `.org`, `.net`, `.io`, `.uk` |
| **Authoritative** | The domain owner's DNS provider | `example.com`'s records |

---

## 3. How Domain Resolution Works (Step by Step)

When you visit `www.example.com`, your machine performs a **DNS lookup**, typically via a **recursive resolver** (often run by your ISP or a public one like `8.8.8.8`):

```
1. Browser/OS checks its CACHE (and hosts file). Found? Done.
2. Otherwise, ask the RECURSIVE RESOLVER (e.g., ISP / 8.8.8.8).
3. Resolver checks its cache. If not cached, it queries the hierarchy:
      a. ROOT server: "Who handles .com?"      -> points to .com TLD servers
      b. TLD server:  "Who handles example.com?"-> points to example.com's authoritative server
      c. AUTHORITATIVE server: "What's www.example.com's IP?" -> returns the A record
4. Resolver caches the answer (per TTL) and returns the IP to your machine.
5. Browser connects to that IP.
```

```
   You -> Recursive Resolver -> Root -> TLD (.com) -> Authoritative (example.com)
                            <- IP address comes back, gets cached along the way
```

- **Recursive query:** your machine asks the resolver to do *all* the work and return a final answer.
- **Iterative query:** the resolver asks each server, getting referrals, until it reaches the authoritative answer.
- Most lookups hit a **cache** long before reaching the root — caching makes DNS fast and scalable.

---

## 4. DNS Record Types

DNS stores different **record types** for a domain. The essentials:

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps a name → **IPv4** address | `example.com → 93.184.216.34` |
| **AAAA** | Maps a name → **IPv6** address | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Alias: maps a name → **another name** | `www.example.com → example.com` |
| **MX** | Mail server for the domain | `example.com → mail.example.com` |
| **TXT** | Arbitrary text (SPF, DKIM, verification) | domain verification, email auth |
| **NS** | Authoritative name servers for the domain | `example.com → ns1.provider.com` |
| **PTR** | Reverse lookup: IP → name | `34.216.184.93.in-addr.arpa → example.com` |
| **SOA** | Start of Authority: zone metadata | serial, refresh, TTL defaults |
| **SRV** | Service location (host + port) | used by some protocols |

### 4.1 A record vs CNAME (a common confusion)
- **A record:** points a name *directly* to an **IP address**.
- **CNAME:** points a name to *another name* (an alias), which is then resolved further.
```
A:      blog.example.com   ->  93.184.216.34       (direct to IP)
CNAME:  www.example.com    ->  example.com         (alias; resolve example.com next)
```
> ⚠️ A CNAME **can't coexist** with other records at the same name, and the **root/apex domain** (`example.com` itself) generally **can't be a CNAME** (providers offer "ALIAS"/"ANAME" or flattening to work around this).

---

## 5. TTL and Caching

**TTL (Time To Live)** is a value (in seconds) attached to each DNS record telling resolvers **how long to cache** the answer before re-querying.

```
example.com  A  93.184.216.34  TTL=3600   (cache for 1 hour)
```

### 5.1 The TTL trade-off
| Low TTL (e.g., 60s) | High TTL (e.g., 86400s / 1 day) |
|---------------------|----------------------------------|
| Faster propagation of changes | Slower to propagate changes |
| More DNS queries (more load) | Fewer queries, better performance |
| Good before a planned migration | Good for stable records |

> **Practical tip:** before changing a record (e.g., migrating servers), **lower the TTL** in advance so the change propagates quickly. Raise it again afterward.

### 5.2 Caching layers
DNS answers are cached at multiple levels — browser → OS → recursive resolver → along the hierarchy. This is why a DNS change can take time to be seen everywhere ("DNS propagation"), bounded by the TTL.

---

## 6. Tools to Inspect DNS

| Tool | Use |
|------|-----|
| `dig example.com` | Detailed DNS query (records, TTL, which server answered) |
| `dig example.com A` / `AAAA` / `MX` | Query a specific record type |
| `nslookup example.com` | Simpler DNS lookup |
| `host example.com` | Quick lookup |
| `/etc/hosts` (or Windows `hosts`) | Local overrides checked *before* DNS |

```bash
dig +short example.com        # just the IP
dig example.com MX            # mail servers
```
(These are part of the Linux toolkit in section 0.3.)

---

## 7. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "DNS changes are instant" | They're bounded by **TTL** caching ("propagation") |
| Using CNAME at the apex/root domain | Usually not allowed — use ALIAS/ANAME |
| A record vs CNAME confusion | A → IP; CNAME → another name |
| "DNS is one big server" | It's distributed & hierarchical with heavy caching |
| Forgetting the `hosts` file overrides DNS | Local `/etc/hosts` wins — handy for testing, easy to forget |
| Ignoring TTL before a migration | Lower TTL beforehand for fast cutover |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Service discovery (Phase 12):** Kubernetes and Consul use **DNS-based** service discovery — services find each other by name.
- **Load balancing & CDNs:** DNS can return different IPs (round-robin, geo-routing) — the basis of global scaling (later note).
- **Deployments/migrations:** TTL management enables zero-downtime cutovers (Phase 10).
- **Debugging "can't connect":** the first question is often "does the name resolve?" — `dig`/`nslookup` are essential.
- **Email/verification:** MX/TXT records (SPF/DKIM) matter for sending transactional emails (Projects 4, 5).
- **Internal DNS** in cloud VPCs resolves private service names (Phase 16.7).

---

## 9. Quick Self-Check Questions

1. What problem does DNS solve, and at which OSI layer does it operate?
2. Describe the DNS hierarchy from root to authoritative.
3. Walk through resolving `www.example.com` step by step.
4. What's the difference between a recursive and an iterative query?
5. What do A, AAAA, CNAME, and MX records do?
6. What's the difference between an A record and a CNAME? Why can't the apex usually be a CNAME?
7. What is TTL, and what's the trade-off between low and high values?
8. Why might a DNS change not be visible everywhere immediately?

---

## 10. Key Terms Glossary

- **DNS:** system mapping domain names → IP addresses.
- **Domain name:** human-readable address (`example.com`).
- **Recursive resolver:** server that performs the full lookup for you.
- **Root / TLD / authoritative server:** levels of the DNS hierarchy.
- **A / AAAA record:** name → IPv4 / IPv6 address.
- **CNAME:** alias from one name to another.
- **MX / TXT / NS / PTR / SOA:** mail / text / name-server / reverse / zone records.
- **TTL:** how long a record may be cached (seconds).
- **DNS propagation:** delay for changes to spread, bounded by TTL.
- **`/etc/hosts`:** local name→IP overrides checked before DNS.
- **`dig` / `nslookup`:** DNS query tools.

---

*Previous topic: **IP Addresses & Subnetting**.*
*Next topic: **TCP vs UDP**.*
