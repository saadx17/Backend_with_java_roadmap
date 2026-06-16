# OSI Model & TCP/IP Model

> **Phase 0 — Computer Science Fundamentals → 0.2 Networking Fundamentals**
> Goal: Understand the two conceptual models that describe how data travels across networks — the 7-layer OSI model and the practical 4-layer TCP/IP model — the mental framework underpinning all networking.

---

## 0. The Big Picture

Networking is **layered**: instead of one giant program handling everything from electrical signals to web pages, the work is split into **layers**, each with a focused responsibility. Each layer talks only to the layers directly above and below it.

> **Why layers?** Separation of concerns. The app developer doesn't worry about voltages on a wire; the network card doesn't worry about HTTP. Each layer can change independently as long as the interface stays stable.

Two models describe this:
- **OSI Model** — a 7-layer *conceptual/teaching* model.
- **TCP/IP Model** — a 4-layer *practical* model that the real internet runs on.

---

## 1. The OSI Model (7 Layers)

**OSI** = Open Systems Interconnection. A reference model dividing networking into **7 layers**, from physical wires (bottom) to applications (top).

```
 7. Application   <- user-facing protocols (HTTP, DNS, FTP)
 6. Presentation  <- encryption, compression, encoding (TLS, JPEG, UTF-8)
 5. Session       <- establishing/managing sessions
 4. Transport     <- end-to-end delivery (TCP, UDP) — ports, reliability
 3. Network       <- routing across networks (IP) — IP addresses
 2. Data Link     <- node-to-node on the same link (Ethernet, MAC, switches)
 1. Physical      <- bits as signals (cables, radio, voltages)
```

> **Mnemonic (top→bottom):** "**A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing"
> (bottom→top): "**P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way"

### 1.1 Each layer's job

| # | Layer | Responsibility | Examples | Data unit |
|---|-------|----------------|----------|-----------|
| 7 | **Application** | Interface to the user/app; high-level protocols | HTTP, HTTPS, DNS, FTP, SMTP | Data |
| 6 | **Presentation** | Translate/format data: encryption, compression, encoding | TLS/SSL, JPEG, ASCII, UTF-8 | Data |
| 5 | **Session** | Open, manage, and close conversations (sessions) | sockets, RPC, session setup | Data |
| 4 | **Transport** | End-to-end delivery between processes; reliability; ports | **TCP**, **UDP** | Segment (TCP) / Datagram (UDP) |
| 3 | **Network** | Logical addressing & routing across networks | **IP**, ICMP, routers | Packet |
| 2 | **Data Link** | Node-to-node transfer on the same physical link; MAC addressing | Ethernet, Wi-Fi, switches, MAC | Frame |
| 1 | **Physical** | Transmit raw bits as electrical/optical/radio signals | cables, fiber, radio, hubs | Bit |

### 1.2 Layer-by-layer detail
- **Layer 1 — Physical:** the actual hardware and signals. Defines voltages, pin layouts, cable types, frequencies. Just moves **bits**.
- **Layer 2 — Data Link:** moves **frames** between two directly-connected nodes. Uses **MAC addresses** (hardware addresses burned into network cards). **Switches** operate here. Handles error detection on the link.
- **Layer 3 — Network:** moves **packets** across *multiple* networks (the internet). Uses **IP addresses** for logical addressing. **Routers** operate here, choosing paths (routing).
- **Layer 4 — Transport:** delivers data between **processes** (via **ports**). **TCP** provides reliable, ordered delivery; **UDP** provides fast, connectionless delivery. (Next notes.)
- **Layer 5 — Session:** establishes, maintains, and tears down communication sessions between applications.
- **Layer 6 — Presentation:** translates data formats — **encryption (TLS)**, compression, character encoding (UTF-8 — recall Phase 1.4).
- **Layer 7 — Application:** the protocols apps use directly — **HTTP**, DNS, SMTP, FTP. (This is *not* the app itself, but the protocols it speaks.)

---

## 2. The TCP/IP Model (4 Layers)

The **TCP/IP model** (a.k.a. the Internet protocol suite) is the **practical** model the real internet uses. It collapses OSI's 7 layers into **4**.

```
TCP/IP (4 layers)              maps to OSI (7 layers)
+------------------------+
| Application            |  =  Application + Presentation + Session (5,6,7)
+------------------------+
| Transport              |  =  Transport (4)         TCP, UDP
+------------------------+
| Internet               |  =  Network (3)           IP
+------------------------+
| Network Access (Link)  |  =  Data Link + Physical (1,2)
+------------------------+
```

| TCP/IP Layer | Role | Protocols |
|--------------|------|-----------|
| **Application** | App-level protocols (combines OSI 5–7) | HTTP, HTTPS, DNS, FTP, SMTP, SSH |
| **Transport** | Process-to-process delivery | TCP, UDP |
| **Internet** | Addressing & routing | IP, ICMP |
| **Network Access** | Physical transmission on the local link | Ethernet, Wi-Fi |

> The **TCP/IP model is what actually runs the internet**; OSI is the richer teaching framework. People often mix them — e.g., "Layer 7" (OSI Application) and "Layer 4" (OSI Transport) are common shorthand even when discussing TCP/IP systems (e.g., L4 vs L7 load balancers — later note).

---

## 3. Encapsulation — How Data Travels Down and Up

As data moves **down** the layers on the sender, each layer **wraps** it with its own header (and the link layer adds a trailer) — this is **encapsulation**. On the receiver, each layer **unwraps** its header as data moves **up** (decapsulation).

```
SENDER (down the stack)                RECEIVER (up the stack)
Application:  [ HTTP data ]                       [ HTTP data ]
Transport:   [TCP hdr|  data  ]                [TCP hdr| data ]
Network:     [IP hdr|TCP| data ]            [IP hdr|TCP| data ]
Data Link:   [Eth|IP|TCP| data |Eth trailer]  ...unwrapped...
Physical:    101010101010... (bits on the wire) -> received as bits
```

- Each layer adds its **own header** with control info for its peer layer on the other side.
- The receiving layer reads/removes *its* header and passes the rest up.
- This is why the same byte stream "means" different things at different layers (recall Phase 0: bits get interpreted by context).

### 3.1 Data unit names by layer (the PDU terms)
| Layer | Protocol Data Unit (PDU) |
|-------|--------------------------|
| Transport (TCP) | **Segment** |
| Transport (UDP) | **Datagram** |
| Network | **Packet** |
| Data Link | **Frame** |
| Physical | **Bit** |

---

## 4. A Concrete Example: Loading a Web Page

When your browser requests `https://example.com`:
```
7/App   : Browser builds an HTTP GET request
6/Pres  : TLS encrypts it; data encoded as UTF-8
5/Sess  : A session/socket is established
4/Trans : TCP splits it into segments, adds port numbers (e.g., dest 443), ensures delivery
3/Net   : IP wraps segments in packets, adds source/dest IP addresses, routes them
2/Link  : Ethernet/Wi-Fi frames carry packets to the next hop (router) using MAC addresses
1/Phys  : Bits travel as electrical/optical/radio signals
... across many routers ...
Server  : Reverses the process up the stack -> web server reads the HTTP request
```
Every concept in the rest of section 0.2 (IP, DNS, TCP/UDP, ports, HTTP, TLS, load balancers) slots into a layer of this picture.

---

## 5. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "OSI is what the internet uses" | The internet runs on **TCP/IP**; OSI is a reference model |
| Confusing MAC (L2) with IP (L3) addresses | MAC = local hardware address; IP = logical, routable address |
| "Switch and router are the same" | Switch = L2 (local); Router = L3 (between networks) |
| "TLS is its own layer" | TLS sits around L6 (presentation) / between transport & app |
| Memorizing layers without their jobs | Understand each layer's *responsibility* |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **L4 vs L7 load balancers** (later note) directly use OSI layer numbers — Transport (4) vs Application (7) routing.
- **Debugging:** knowing which layer a problem lives in (DNS? TCP? HTTP? TLS?) is the core skill for diagnosing connectivity issues in production.
- **HTTP, TLS, ports, IP, DNS** — every backend concept in this section maps to a layer.
- **Performance:** understanding encapsulation overhead and where latency accrues (handshakes, routing hops) informs optimization (Phase 13).
- **Security:** each layer has its own attack surface and defenses (Phase 15).

---

## 7. Quick Self-Check Questions

1. Why are networks organized into layers?
2. List the 7 OSI layers in order and one responsibility of each.
3. How do the 4 TCP/IP layers map to the 7 OSI layers?
4. Which model does the real internet use?
5. What is encapsulation, and what does each layer add?
6. Name the PDU (data unit) at the transport, network, data link, and physical layers.
7. What's the difference between a switch (L2) and a router (L3)?
8. At which layer do IP addresses operate? Ports? MAC addresses?

---

## 8. Key Terms Glossary

- **OSI model:** 7-layer conceptual networking reference model.
- **TCP/IP model:** 4-layer practical model the internet uses.
- **Layer:** a level of networking with a focused responsibility.
- **Encapsulation:** wrapping data with headers as it moves down the stack.
- **PDU (Protocol Data Unit):** the data unit name per layer (segment/packet/frame/bit).
- **MAC address:** hardware/link-layer (L2) address.
- **IP address:** logical network-layer (L3) address.
- **Port:** transport-layer (L4) process identifier.
- **Switch / Router:** L2 (local) / L3 (inter-network) devices.
- **Header / Trailer:** control info added by a layer.

---

*This is the first note of **Section 0.2 — Networking Fundamentals**.*
*Next topic: **IP Addresses (IPv4, IPv6, Subnetting)**.*
