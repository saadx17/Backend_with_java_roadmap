# Ports, Sockets, Firewalls, Proxies, Load Balancers & CDNs

> **Phase 0 — Computer Science Fundamentals → 0.2 Networking Fundamentals**
> Goal: Understand the network infrastructure that sits between clients and your application — ports & sockets, firewalls, proxies (forward vs reverse), load balancers (L4 vs L7), and CDNs.

---

## 0. The Big Picture

Between a user's browser and your application code sits a chain of infrastructure. This note covers the pieces you'll deploy behind, configure, and debug:

```
Client -> [CDN] -> [Firewall] -> [Load Balancer / Reverse Proxy] -> [Your App servers]
```

Understanding these explains how traffic actually reaches your Spring app — and how to scale, secure, and speed it up.

---

## 1. Ports & Sockets

### 1.1 Ports
An **IP address** identifies a *machine*; a **port** identifies a specific *process/service* on it (Transport layer, recall TCP/UDP note). A port is a 16-bit number (0–65535).
```
93.184.216.34 : 443
   IP address    port    -> "the HTTPS service on this machine"
```

| Range | Name | Use |
|-------|------|-----|
| 0–1023 | **Well-known ports** | Standard services (need privileges to bind) |
| 1024–49151 | **Registered ports** | App-assigned (e.g., 8080) |
| 49152–65535 | **Ephemeral ports** | Temporary client-side ports |

### 1.2 Common well-known ports (memorize these)
| Port | Service |
|------|---------|
| 20/21 | FTP |
| 22 | SSH / SCP |
| 25 | SMTP (email) |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8080 | HTTP (common app/dev default, e.g., Spring Boot) |
| 27017 | MongoDB |
| 9092 | Kafka |

### 1.3 Sockets
A **socket** is one endpoint of a two-way connection, uniquely identified by the **4-tuple**:
```
(source IP, source port, destination IP, destination port)
```
This 4-tuple is what lets a server handle thousands of simultaneous connections on the *same* port (e.g., 443) — each connection has a distinct client IP/port combination.

> Sockets are the OS-level API your app uses for networking (Phase 0 I/O note's blocking/non-blocking models apply here). Java exposes `Socket`/`ServerSocket` and NIO channels.

---

## 2. Firewalls

A **firewall** controls network traffic based on rules — deciding what's **allowed** or **blocked** by IP, port, protocol, or direction.

```
Internet ----[ Firewall: allow 443, block everything else ]----> Server
```

| Concept | Meaning |
|---------|---------|
| **Inbound rules** | What traffic can come *in* (e.g., allow 443 from anywhere) |
| **Outbound rules** | What traffic can go *out* |
| **Default deny** | Block everything not explicitly allowed (best practice) |
| **Stateful firewall** | Tracks connections; allows return traffic automatically |

> In the cloud, firewalls appear as **security groups / network ACLs** (AWS), controlling which ports/IPs can reach your servers (Phase 16.7). A classic bug: "my service is up but unreachable" → a firewall/security-group rule is blocking the port.

---

## 3. Proxies: Forward vs Reverse

A **proxy** is an intermediary that forwards requests. The direction it serves defines the type.

### 3.1 Forward proxy (sits in front of *clients*)
Acts **on behalf of clients**, hiding them from the server:
```
[Client] -> [Forward Proxy] -> Internet -> [Server]
            (server sees the proxy, not the client)
```
Uses: corporate web filtering, anonymity/VPN-like access, caching outbound requests, bypassing geo-restrictions.

### 3.2 Reverse proxy (sits in front of *servers*)
Acts **on behalf of servers**, hiding them from clients:
```
[Client] -> Internet -> [Reverse Proxy] -> [App Server 1]
                                        -> [App Server 2]
            (client sees the proxy, not the real servers)
```
Uses (very common in backend deployments):
| Reverse proxy job | Benefit |
|-------------------|---------|
| **Load balancing** | Distribute traffic across servers |
| **TLS termination** | Decrypt HTTPS once at the edge (recall TLS note) |
| **Caching** | Serve cached responses |
| **Compression** | gzip/brotli responses |
| **Security** | Hide backend topology; filter/rate-limit |
| **Routing** | Map paths/hosts to different backends |

> **Nginx**, **HAProxy**, **Envoy**, and cloud load balancers are common reverse proxies. Your Spring app typically runs *behind* one.

### 3.3 Forward vs reverse at a glance
| | Forward proxy | Reverse proxy |
|---|---------------|---------------|
| Acts for | The **client** | The **server** |
| Hidden party | Client (from server) | Server (from client) |
| Typical use | Filtering, anonymity, outbound caching | Load balancing, TLS, caching, routing |

---

## 4. Load Balancers (L4 vs L7)

A **load balancer (LB)** distributes incoming traffic across multiple servers to improve **scalability**, **availability**, and **performance**. It's a specialized reverse proxy.

```
            -> [Server 1]
[Clients] -> [ LB ] -> [Server 2]    (spreads load; routes around failures)
            -> [Server 3]
```

### 4.1 Why load balance
- **Horizontal scaling:** add more servers to handle more traffic (Phase 13).
- **High availability:** if a server dies, the LB stops sending it traffic (health checks).
- **No single point of overload.**

### 4.2 L4 vs L7 (the key distinction — uses OSI layer numbers!)
| | **L4 load balancer** | **L7 load balancer** |
|---|----------------------|----------------------|
| OSI layer | Transport (4) — TCP/UDP | Application (7) — HTTP |
| Routes based on | IP + port | URL path, headers, cookies, host |
| Sees content? | No (just packets) | Yes (reads HTTP) |
| Speed | Faster (less inspection) | Slightly slower (parses HTTP) |
| Capabilities | Simple, fast distribution | Path routing, SSL termination, header-based routing |
| Example | AWS NLB | AWS ALB, Nginx, Envoy |

> **L4** balances connections without understanding them; **L7** understands HTTP and can route `/api` to one service and `/images` to another, do TLS termination, sticky sessions, etc. Microservice **API gateways** are L7 (Phase 12.4).

### 4.3 Load balancing algorithms
| Algorithm | How it picks a server |
|-----------|------------------------|
| **Round-robin** | Each server in turn |
| **Least connections** | The server with the fewest active connections |
| **IP hash** | Based on client IP (→ same client → same server, "sticky") |
| **Weighted** | Proportional to server capacity |

### 4.4 Health checks & statelessness
- LBs run **health checks** (e.g., hit `/health`) and route only to healthy servers (liveness/readiness — Phase 9, 10).
- For load balancing to work well, app servers should be **stateless** (no in-memory session tied to one server) — store session/state externally (Redis) so any server can handle any request (Phase 13).

---

## 5. CDNs (Content Delivery Networks)

A **CDN** is a globally distributed network of **edge servers** that cache content **close to users**, reducing latency and offloading your origin server.

```
User in Tokyo  -> nearest CDN edge (Tokyo) -> cached content (fast!)
                                  (only on cache miss) -> Origin server (e.g., US)
```

### 5.1 How a CDN works
1. User requests an asset (image, JS, CSS, video).
2. The request goes to the **nearest edge server** (via DNS/anycast routing).
3. **Cache hit** → served instantly from the edge.
4. **Cache miss** → edge fetches from your **origin**, caches it (per TTL — recall DNS/HTTP caching), serves it.

### 5.2 Benefits
| Benefit | Why |
|---------|-----|
| **Lower latency** | Content is geographically close to users |
| **Reduced origin load** | Edges absorb most requests |
| **Scalability** | Handles traffic spikes / flash crowds |
| **DDoS protection** | Distributes and absorbs attack traffic |
| **Bandwidth savings** | Less traffic to your origin |

### 5.3 What CDNs serve
- Best for **static assets** (images, CSS, JS, fonts, video) and cacheable responses.
- Modern CDNs also do **edge compute**, API caching, and TLS termination.
- Examples: Cloudflare, Akamai, AWS CloudFront, Fastly.

> CDNs apply the **caching + locality** principle (Phase 0 memory-hierarchy note) at global scale: keep frequently used data close to where it's needed.

---

## 6. Putting It All Together (a typical request path)

```
Browser
  -> DNS resolves the name (DNS note)
  -> CDN edge (cached static assets served here)
  -> [cache miss / dynamic request] travels to your origin region
  -> Firewall / security group (allow 443)
  -> L7 Load Balancer / Reverse Proxy (TLS termination, routing, rate limiting)
  -> One of several stateless App servers (your Spring Boot app on port 8080)
  -> App talks to DB (5432), Redis (6379), etc. over their ports
```
Every concept in section 0.2 appears in this path.

---

## 7. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "Service is up so it's reachable" | A firewall/security group may block the port |
| Confusing forward vs reverse proxy | Forward = for clients; reverse = for servers |
| "Load balancer = just round-robin" | Many algorithms; L4 vs L7 differ greatly |
| Stateful app servers behind an LB | Breaks scaling/failover — externalize session state |
| "CDN caches everything" | Mainly cacheable/static content; respect cache headers |
| Binding app to `127.0.0.1` behind an LB | Won't receive external traffic — bind `0.0.0.0` |
| Forgetting health checks | LB may route to dead servers |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Deployment (Phase 10):** your app runs behind a reverse proxy/LB; you expose port 8080 internally, 443 externally.
- **TLS termination** at the LB means your app often speaks plain HTTP internally (TLS note).
- **Statelessness + external session store (Redis)** is required for horizontal scaling behind an LB (Phase 5.7, 13).
- **API Gateway (Phase 12.4)** is an L7 reverse proxy doing routing, auth, and rate limiting for microservices.
- **Health endpoints** (`/actuator/health`) feed LB/K8s probes (Phase 9, 10).
- **CDN** offloads static assets and protects the origin (Phase 13, 16).
- **Firewalls/security groups** control which ports services can reach (Phase 16.7) — a top cause of "can't connect" issues.

---

## 9. Quick Self-Check Questions

1. What's the difference between an IP address and a port?
2. What 4-tuple uniquely identifies a socket, and why does it allow many connections on port 443?
3. Name the standard ports for HTTP, HTTPS, SSH, PostgreSQL, and Redis.
4. What does a firewall do, and what is "default deny"?
5. What's the difference between a forward and a reverse proxy?
6. What's the difference between an L4 and an L7 load balancer?
7. Name three load balancing algorithms.
8. Why must app servers be stateless behind a load balancer?
9. How does a CDN reduce latency, and what does it best serve?

---

## 10. Key Terms Glossary

- **Port:** transport-layer identifier of a process (0–65535).
- **Well-known/ephemeral ports:** standard services / temporary client ports.
- **Socket:** a connection endpoint; the (srcIP, srcPort, dstIP, dstPort) tuple.
- **Firewall:** rule-based traffic filter (security group in the cloud).
- **Forward proxy:** intermediary acting for clients.
- **Reverse proxy:** intermediary acting for servers.
- **Load balancer:** distributes traffic across servers.
- **L4 / L7 LB:** transport-level (IP/port) vs application-level (HTTP) routing.
- **Round-robin / least-connections / IP hash:** LB algorithms.
- **Health check:** probe to route only to healthy servers.
- **Stateless server:** holds no per-client state (enables scaling).
- **CDN:** geographically distributed cache of edge servers.
- **Edge server / origin:** CDN cache node / your source server.
- **TLS termination:** decrypting HTTPS at a proxy/LB.

---

*Previous topic: **HTTPS & the TLS Handshake**.*
*Next topic: **WebSockets, gRPC/Protocol Buffers & CORS**.*
