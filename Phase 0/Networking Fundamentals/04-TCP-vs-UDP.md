# TCP vs UDP

> **Phase 0 — Computer Science Fundamentals → 0.2 Networking Fundamentals**
> Goal: Master the two transport-layer protocols — TCP (reliable, connection-oriented) and UDP (fast, connectionless) — the handshake, reliability mechanisms, and when to use each.

---

## 0. The Big Picture

**TCP** and **UDP** are the two main **Transport layer (OSI L4)** protocols. They deliver data between **processes** (identified by **ports**) on top of IP (which only delivers between *machines*).

| | **TCP** | **UDP** |
|---|---------|---------|
| Full name | Transmission Control Protocol | User Datagram Protocol |
| Connection | **Connection-oriented** (handshake first) | **Connectionless** (just send) |
| Reliability | **Reliable** (guaranteed, ordered, no dups) | **Unreliable** (best-effort) |
| Ordering | Preserves order | No ordering guarantee |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Use cases | Web, APIs, email, file transfer, DB | Streaming, gaming, VoIP, DNS |

> **The core trade-off:** TCP guarantees correctness at the cost of overhead; UDP is fast and lean but makes no guarantees. (This mirrors the blocking/non-blocking trade-off theme from Phase 0.)

---

## 1. TCP — Transmission Control Protocol

TCP provides a **reliable, ordered, error-checked** byte stream between two endpoints. It's the workhorse of the web and almost all backend APIs.

### 1.1 Connection-oriented: the 3-way handshake
Before any data flows, TCP **establishes a connection** with a 3-step handshake:
```
Client                         Server
  |  --- SYN (seq=x) -------->  |   "Let's connect; my seq is x"
  |  <-- SYN-ACK (seq=y,ack=x+1)|   "OK; my seq is y, I got yours"
  |  --- ACK (ack=y+1) ------>  |   "Got it; connection established"
  |                             |
  | <======= data flows ======> |
```
| Step | Packet | Meaning |
|------|--------|---------|
| 1 | **SYN** | Client requests a connection (synchronize) |
| 2 | **SYN-ACK** | Server acknowledges and synchronizes back |
| 3 | **ACK** | Client acknowledges → connection established |

> This handshake is why TCP has higher **latency** to start — a full round-trip before data. (TLS adds *more* round-trips on top — TLS note.)

### 1.2 Connection teardown (4-way)
Closing is a 4-step process (FIN/ACK from each side) so both ends agree the conversation is over.

### 1.3 How TCP guarantees reliability
| Mechanism | What it does |
|-----------|--------------|
| **Sequence numbers** | Every byte is numbered → receiver can **reorder** and detect gaps |
| **Acknowledgments (ACK)** | Receiver confirms what it got; unacknowledged data is **retransmitted** |
| **Retransmission** | Lost segments are resent (after timeout / duplicate ACKs) |
| **Checksums** | Detect corrupted data → discard & retransmit |
| **Flow control** (sliding window) | Receiver advertises how much it can accept → sender won't overwhelm it |
| **Congestion control** | Sender slows down when the network is congested (avoids collapse) |

> These mechanisms together give the "it just works" reliability of TCP — at the cost of headers, ACKs, and round-trips.

### 1.4 TCP is a byte stream (no message boundaries)
TCP delivers an ordered **stream of bytes**, not discrete messages. The application must define its own framing (e.g., HTTP uses `Content-Length`/chunking) — a sender's two `write()`s may arrive as one read, or split. (Relevant when writing low-level socket code.)

---

## 2. UDP — User Datagram Protocol

UDP is a **connectionless**, **best-effort** protocol. It just sends independent **datagrams** with minimal overhead — no handshake, no acknowledgments, no ordering.

```
Client                         Server
  |  --- datagram 1 -------->   |    (no connection setup; just send)
  |  --- datagram 2 -------->   |    (may arrive out of order...)
  |  --- datagram 3 --X         |    (...or not arrive at all — no retransmit)
```

### 2.1 Characteristics
- **No handshake** → no startup latency; send immediately.
- **No reliability** → packets may be lost, duplicated, or arrive out of order.
- **No flow/congestion control** (basic UDP) → the app must handle it if needed.
- **Preserves message boundaries** → each datagram is a discrete unit (unlike TCP's stream).
- **Lower overhead** → smaller header (8 bytes vs TCP's 20+).

### 2.2 Why use something "unreliable"?
For real-time data, **speed beats perfection**:
- In a video call, a dropped frame is better than pausing to resend an old one.
- The app can add *just* the reliability it needs (e.g., **QUIC**, the protocol behind **HTTP/3**, builds reliability on top of UDP).

---

## 3. When to Use Which

### 3.1 Use TCP when...
- **Data integrity is essential** — every byte must arrive, in order.
- Examples: **HTTP/HTTPS (web & REST APIs)**, email (SMTP/IMAP), file transfer (FTP/SFTP), SSH, **database connections**, gRPC.

### 3.2 Use UDP when...
- **Speed/low latency matters more than perfect delivery**, or you do your own reliability.
- Examples: **DNS** lookups (small, fast), **video/audio streaming**, **online gaming**, **VoIP**, real-time telemetry, **QUIC/HTTP/3**.

### 3.3 Decision table
| Need | Choose |
|------|--------|
| Guaranteed, ordered delivery | **TCP** |
| Lowest latency, tolerate loss | **UDP** |
| Web/API/DB traffic | **TCP** |
| Real-time media / gaming | **UDP** |
| Small request-response (e.g., DNS) | **UDP** (with TCP fallback) |

---

## 4. TCP vs UDP — Side by Side

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Yes (3-way handshake) | No |
| Reliable delivery | Yes | No |
| Ordering | Yes | No |
| Duplicate protection | Yes | No |
| Flow/congestion control | Yes | No (app's job) |
| Message boundaries | No (byte stream) | Yes (datagrams) |
| Header size | 20+ bytes | 8 bytes |
| Speed | Slower | Faster |
| Overhead | Higher | Lower |

---

## 5. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "UDP is always better because it's faster" | Only when loss is acceptable / you add your own reliability |
| "TCP guarantees data is *correct*" | It guarantees delivery/order/integrity of bytes, not that the *content* is meaningful |
| "TCP preserves message boundaries" | No — it's a **byte stream**; you must frame messages |
| "Handshake is negligible" | It's a round-trip of latency before any data (matters at scale/distance) |
| "DNS is always UDP" | UDP normally, but falls back to **TCP** for large responses |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **HTTP/REST/gRPC run over TCP** — virtually all your backend traffic. Understanding the handshake explains **connection latency** and why **connection pooling/keep-alive** matters (Phase 4 HikariCP, Phase 13).
- **TLS handshake** adds round-trips on top of TCP — motivates HTTP/2 multiplexing and connection reuse (TLS & HTTP notes).
- **WebSockets** upgrade an existing TCP connection for persistent bidirectional comms (later note).
- **HTTP/3 (QUIC) over UDP** is the emerging standard — faster connection setup, no head-of-line blocking.
- **Latency budgeting (Phase 13/16):** each TCP/TLS round-trip costs real milliseconds — fewer round-trips = faster.
- **Databases** use long-lived TCP connections → pool them rather than reconnecting per query.

---

## 7. Quick Self-Check Questions

1. At which OSI layer do TCP and UDP operate, and what do ports identify?
2. Describe the TCP 3-way handshake and why it adds latency.
3. List four mechanisms TCP uses to guarantee reliability.
4. Why is TCP a "byte stream," and what must apps do about message boundaries?
5. What guarantees does UDP make? Why use it despite being "unreliable"?
6. Give three use cases each for TCP and UDP.
7. Why does DNS sometimes use TCP instead of UDP?
8. What protocol does HTTP/3 use, and why?

---

## 8. Key Terms Glossary

- **TCP:** reliable, connection-oriented transport protocol.
- **UDP:** fast, connectionless, best-effort transport protocol.
- **Port:** transport-layer identifier of a process/service.
- **3-way handshake:** SYN → SYN-ACK → ACK connection setup.
- **SYN / ACK / FIN:** synchronize / acknowledge / finish flags.
- **Sequence number / acknowledgment:** byte numbering / receipt confirmation.
- **Retransmission:** resending lost segments.
- **Flow control:** receiver-paced sending (sliding window).
- **Congestion control:** network-aware rate adjustment.
- **Datagram:** an independent UDP message.
- **Byte stream:** TCP's ordered, boundary-less data delivery.
- **QUIC / HTTP/3:** reliable protocol built on UDP.

---

*Previous topic: **DNS**.*
*Next topic: **HTTP Protocol Deep Dive**.*
