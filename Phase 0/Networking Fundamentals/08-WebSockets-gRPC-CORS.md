# WebSockets, gRPC/Protocol Buffers & CORS

> **Phase 0 — Computer Science Fundamentals → 0.2 Networking Fundamentals**
> Goal: Understand three modern web-communication topics — WebSockets (persistent connections), gRPC & Protocol Buffers (efficient RPC), and CORS (cross-origin security) — at an awareness level that prepares you for later phases.

---

## 0. The Big Picture

Plain HTTP request–response (recall HTTP note) doesn't fit every need:
- It can't easily **push** data from server to client → **WebSockets**.
- Its text-based JSON is verbose for high-performance service-to-service calls → **gRPC + Protocol Buffers**.
- Browsers restrict cross-origin requests for security → **CORS** governs when they're allowed.

This note covers each so the concepts aren't new when they reappear in Phases 11, 12, 16.

---

## 1. WebSockets (Persistent, Bidirectional Connections)

### 1.1 The problem with HTTP for real-time
HTTP is **request–response**: the client must *ask* for data; the server can't initiate. For real-time features (chat, live feeds, notifications), workarounds were clumsy:
| Technique | How | Drawback |
|-----------|-----|----------|
| **Polling** | Client repeatedly asks "any updates?" | Wasteful, laggy |
| **Long polling** | Server holds the request open until data is ready | Better, still heavy |
| **WebSockets** | A persistent, two-way connection | The real solution |

### 1.2 What WebSockets are
A **WebSocket** is a **persistent, full-duplex** (two-way) connection over a single TCP connection. Once established, **both** client and server can send messages **anytime**, with low overhead.

```
HTTP:       Client --request--> Server --response--> Client  (one-shot, client-initiated)
WebSocket:  Client <===== open persistent channel =====> Server
            (either side can push messages at will)
```

### 1.3 How it starts: the upgrade handshake
A WebSocket connection **starts as HTTP**, then **upgrades**:
```
Client request:  GET /chat HTTP/1.1
                 Upgrade: websocket
                 Connection: Upgrade
                 ...
Server response: HTTP/1.1 101 Switching Protocols     <- recall status code 101!
                 Upgrade: websocket
```
After the `101 Switching Protocols` response, the same TCP connection becomes a WebSocket — no more HTTP request/response, just messages (frames) flowing both ways. The URL scheme is `ws://` (or `wss://` for secure, over TLS).

### 1.4 Use cases
- **Chat / messaging**
- **Live notifications / activity feeds**
- **Real-time dashboards** (metrics, prices)
- **Collaborative editing** (Google-Docs-style)
- **Multiplayer games**

### 1.5 Scaling challenges (awareness)
- Each client holds an **open connection** → servers must manage many concurrent connections (recall Phase 0 I/O: non-blocking/event-loop models, the C10K problem).
- **Stateful by nature** → harder to load-balance (need sticky sessions or a shared pub/sub backplane like Redis to broadcast across servers).

---

## 2. gRPC & Protocol Buffers (Efficient RPC)

### 2.1 What gRPC is
**gRPC** is a high-performance **RPC (Remote Procedure Call)** framework by Google. Instead of "send an HTTP request to a URL," you **call a method on a remote service** as if it were local:
```java
// feels like a local method call, but runs on a remote server:
UserResponse user = userServiceStub.getUser(GetUserRequest.newBuilder().setId(42).build());
```
- Runs over **HTTP/2** (multiplexing, streaming — recall HTTP note).
- Uses **Protocol Buffers** for serialization (binary, compact).
- Strongly typed via a **service definition** (`.proto` file).

### 2.2 Protocol Buffers (Protobuf)
**Protobuf** is a **binary** serialization format — a compact, fast, schema-defined alternative to JSON. You define your data and services in a `.proto` file:
```proto
syntax = "proto3";

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}
```
A compiler generates client/server code from this schema in many languages.

### 2.3 gRPC/Protobuf vs REST/JSON
| Aspect | gRPC + Protobuf | REST + JSON |
|--------|-----------------|-------------|
| Format | Binary (compact) | Text (human-readable) |
| Transport | HTTP/2 | Usually HTTP/1.1 |
| Schema | Required (`.proto`) | Optional (OpenAPI) |
| Speed/size | Faster, smaller | Slower, larger |
| Streaming | Built-in (bidirectional) | Limited |
| Browser support | Limited (needs gRPC-Web) | Native |
| Human-debuggable | Harder (binary) | Easy (curl/browser) |
| Best for | **Service-to-service** (microservices) | **Public/web APIs** |

### 2.4 The four gRPC call types
- **Unary:** one request → one response (like normal REST).
- **Server streaming:** one request → a stream of responses.
- **Client streaming:** a stream of requests → one response.
- **Bidirectional streaming:** both stream simultaneously.

> **When to use gRPC:** internal **microservice-to-microservice** communication where performance and strong typing matter (Phase 12.2). **When to use REST:** public APIs, browser clients, human-debuggable endpoints (Phase 5, 7).

---

## 3. CORS (Cross-Origin Resource Sharing)

### 3.1 The problem: the Same-Origin Policy
Browsers enforce the **Same-Origin Policy (SOP)**: by default, JavaScript on one **origin** can't read responses from a *different* origin. An **origin** = **scheme + host + port**:
```
https://app.example.com:443
  ^scheme   ^host        ^port    -> all three must match to be "same origin"
```
| URL | Same origin as `https://app.example.com`? |
|-----|-------------------------------------------|
| `https://app.example.com/page` | ✅ Yes |
| `http://app.example.com` | ❌ No (different scheme) |
| `https://api.example.com` | ❌ No (different host) |
| `https://app.example.com:8080` | ❌ No (different port) |

> SOP is a **security** feature: it stops a malicious site from reading your bank's data using your logged-in session.

### 3.2 What CORS does
**CORS** is a mechanism that lets a server **explicitly allow** specific cross-origin requests by sending special **response headers**. It *relaxes* SOP in a controlled way.

```
Browser JS at https://app.example.com  ---requests--->  https://api.example.com
The API responds with:
   Access-Control-Allow-Origin: https://app.example.com   <- "I permit this origin"
Browser sees the header -> allows the JS to read the response.
(No such header -> browser BLOCKS the response from the JS.)
```

### 3.3 Key CORS response headers
| Header | Purpose |
|--------|---------|
| `Access-Control-Allow-Origin` | Which origin(s) may access the resource (`*` = any) |
| `Access-Control-Allow-Methods` | Allowed HTTP methods (GET, POST, ...) |
| `Access-Control-Allow-Headers` | Allowed request headers |
| `Access-Control-Allow-Credentials` | Whether cookies/auth may be sent |
| `Access-Control-Max-Age` | How long to cache the preflight result |

### 3.4 Preflight requests
For "non-simple" requests (e.g., `PUT`/`DELETE`, custom headers, JSON content type), the browser first sends an **OPTIONS** "preflight" request (recall the OPTIONS method) to ask permission:
```
1. Browser -> OPTIONS /api/users  (preflight: "can I do a PUT with these headers?")
2. Server  -> 200 with Access-Control-Allow-* headers ("yes")
3. Browser -> the actual PUT request
```

### 3.5 Crucial clarifications (very commonly misunderstood)
| Misconception | Reality |
|---------------|---------|
| "CORS is a server security feature" | CORS is enforced by the **browser**, not the server. It protects *users*, not your API |
| "A CORS error means my server rejected it" | The **browser** blocked the JS from reading the response; the request may have reached the server |
| "Set `Access-Control-Allow-Origin: *` to fix it" | `*` can't be combined with credentials and is often too permissive |
| "CORS protects my API from attackers" | It doesn't — non-browser clients (curl, servers) ignore CORS entirely. Use real auth (Phase 15) |

> **Bottom line:** CORS is a **browser-enforced relaxation of the Same-Origin Policy**, configured via server response headers. It's about what *browser JavaScript* may read cross-origin — not API security.

---

## 4. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using polling where WebSockets fit | Use WebSockets for real-time push |
| Forgetting WebSockets are stateful | Plan for sticky sessions / pub-sub when scaling |
| Using gRPC for public/browser APIs | Use REST/JSON; gRPC for internal services |
| Treating Protobuf like JSON | It's binary & schema-driven — needs `.proto` + codegen |
| Thinking CORS secures your API | It's browser-side; use authentication/authorization |
| Slapping `Allow-Origin: *` with credentials | Not allowed together; scope origins explicitly |
| Confusing a CORS error with a server rejection | The browser blocks the read; check headers |

---

## 5. Connection to Backend / Spring (Why This Matters Later)

- **WebSockets (Phase 16.3):** Spring WebSocket + STOMP for chat/notifications/live dashboards; scaling needs a broker (Redis/RabbitMQ).
- **gRPC (Phase 12.2):** idiomatic for **microservice-to-microservice** calls; Spring supports gRPC and Protobuf.
- **CORS (Phase 5.3):** configured in Spring via `@CrossOrigin` or a global `WebMvcConfigurer`/Security config — essential when a frontend (React/Angular) on a different origin calls your API.
- **HTTP/2 & streaming** (gRPC) tie back to the HTTP-versions discussion (performance, Phase 13).
- **Same-Origin Policy + auth** are core web-security concepts (Phase 15) — CORS, CSRF, and cookies interrelate.

---

## 6. Quick Self-Check Questions

1. Why doesn't plain HTTP suit real-time server→client updates?
2. What is a WebSocket, and how does the upgrade handshake work (which status code)?
3. What scaling challenge do WebSockets introduce?
4. What is gRPC, and what does it use for serialization and transport?
5. How does Protobuf differ from JSON? When choose gRPC vs REST?
6. What are the four gRPC call types?
7. What is an "origin," and what does the Same-Origin Policy prevent?
8. What does CORS do, who enforces it, and why doesn't it secure your API?
9. What triggers a CORS preflight, and which method does it use?

---

## 7. Key Terms Glossary

- **WebSocket:** persistent, full-duplex connection over one TCP connection.
- **Full-duplex:** both sides can send simultaneously.
- **Upgrade handshake / `101 Switching Protocols`:** HTTP→WebSocket transition.
- **`ws://` / `wss://`:** WebSocket / secure WebSocket schemes.
- **Polling / long polling:** repeatedly/held HTTP requests for updates.
- **gRPC:** high-performance RPC framework over HTTP/2.
- **RPC:** calling a remote method as if local.
- **Protocol Buffers (Protobuf):** compact binary, schema-defined serialization.
- **`.proto` file:** the schema defining messages and services.
- **Unary / streaming RPC:** single vs streamed request/response patterns.
- **Origin:** scheme + host + port.
- **Same-Origin Policy (SOP):** browser rule restricting cross-origin reads.
- **CORS:** server-declared, browser-enforced relaxation of SOP.
- **Preflight (OPTIONS):** permission check before non-simple cross-origin requests.

---

*Previous topic: **Ports, Sockets, Firewalls, Proxies, Load Balancers & CDNs**.*
*This completes **Section 0.2 — Networking Fundamentals**.*
*Next section in roadmap: **0.3 Operating System Basics**.*
