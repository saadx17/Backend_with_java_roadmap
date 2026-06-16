# IP Addresses (IPv4, IPv6, Subnetting Basics)

> **Phase 0 — Computer Science Fundamentals → 0.2 Networking Fundamentals**
> Goal: Understand IP addressing — IPv4 vs IPv6, public vs private addresses, and subnetting basics (CIDR) — the Layer-3 logical addressing that routes data across the internet.

---

## 0. The Big Picture

An **IP address** is a logical, routable address that identifies a device on a network — the "postal address" of the internet. It operates at the **Network layer (OSI L3)** and is what **routers** use to forward packets across networks (recall the OSI note).

> IP address = *where* a device is on the network. (A **port**, next notes, identifies *which process* on that device.)

Two versions coexist:
- **IPv4** — 32-bit, the classic (running out of addresses).
- **IPv6** — 128-bit, the future (vastly larger space).

---

## 1. IPv4

A **32-bit** address, written as four decimal **octets** (0–255) separated by dots (**dotted-decimal**):
```
192.168.1.10
 |   |   | |
 each octet = 8 bits (0-255)
```
- 32 bits → **2³² ≈ 4.3 billion** addresses (recall Phase 0: 32-bit range).
- Each octet is one byte; underneath it's just binary: `192.168.1.10` = `11000000.10101000.00000001.00001010`.

### 1.1 Network vs host portion
An IPv4 address splits into a **network part** (which network) and a **host part** (which device on it). A **subnet mask** marks the boundary:
```
IP:          192.168.1.10
Subnet mask: 255.255.255.0   (255.255.255 = network, 0 = host)
                  network  | host
```
- `255` (binary `11111111`) = network bits; `0` = host bits.
- Here: `192.168.1` is the network, `.10` is the host.

### 1.2 IPv4 exhaustion
4.3 billion sounds like a lot, but with billions of devices it **ran out**. Workarounds: **NAT** (private addresses sharing one public IP, §3) and the long migration to **IPv6**.

---

## 2. IPv6

A **128-bit** address, written as eight groups of four hex digits separated by colons:
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```
- 128 bits → **2¹²⁸ ≈ 3.4 × 10³⁸** addresses — effectively unlimited (enough for every grain of sand many times over).
- Solves IPv4 exhaustion; also simplifies routing and has built-in features (no NAT needed, optional IPsec).

### 2.1 Shorthand rules
IPv6 addresses can be abbreviated:
- **Drop leading zeros** in each group: `0db8` → `db8`, `0000` → `0`.
- **Collapse one run of all-zero groups** to `::` (only once per address):
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
2001:db8:85a3::8a2e:370:7334            (collapsed)

::1        = loopback (equivalent to IPv4 127.0.0.1)
::         = all zeros (unspecified address)
```

### 2.2 IPv4 vs IPv6 at a glance
| Aspect | IPv4 | IPv6 |
|--------|------|------|
| Size | 32-bit | 128-bit |
| Notation | Dotted decimal (`192.168.1.1`) | Hex, colon-separated |
| Address count | ~4.3 billion | ~3.4 × 10³⁸ |
| Loopback | `127.0.0.1` | `::1` |
| NAT needed? | Yes (scarcity) | No (abundance) |
| Config | Manual/DHCP | Auto-configuration supported |

---

## 3. Public vs Private Addresses & NAT

### 3.1 Private address ranges
Certain IPv4 ranges are reserved for **private networks** (not routable on the public internet). Your home/office devices use these:
| Range | CIDR | Typical use |
|-------|------|-------------|
| `10.0.0.0 – 10.255.255.255` | `10.0.0.0/8` | Large networks |
| `172.16.0.0 – 172.31.255.255` | `172.16.0.0/12` | Medium networks |
| `192.168.0.0 – 192.168.255.255` | `192.168.0.0/16` | Home/small networks |

### 3.2 NAT (Network Address Translation)
**NAT** lets many devices with private IPs share **one public IP** (your router's). The router rewrites addresses/ports so internal devices can reach the internet:
```
[Laptop 192.168.1.10] \
[Phone  192.168.1.11]  ->  [Router: public IP 203.0.113.5]  ->  Internet
[TV     192.168.1.12] /        (NAT translates private <-> public)
```
> NAT is a major reason IPv4 survived address exhaustion — and why your laptop's IP (`192.168.x.x`) differs from the IP websites see (your public IP).

### 3.3 Special addresses
| Address | Meaning |
|---------|---------|
| `127.0.0.1` (`localhost`) | **Loopback** — this same machine |
| `0.0.0.0` | "All interfaces" (bind a server to every address) |
| `255.255.255.255` | Broadcast (all hosts on the local network) |
| `169.254.x.x` | Link-local (auto-assigned when DHCP fails) |

---

## 4. Subnetting Basics & CIDR

**Subnetting** divides a large network into smaller **subnetworks** for organization, security, and efficient address use.

### 4.1 CIDR notation
**CIDR** (Classless Inter-Domain Routing) appends `/n` to an address, where `n` = number of **network bits**:
```
192.168.1.0/24
              ^ 24 network bits, so 32-24 = 8 host bits
```
- `/24` → 24 network bits, 8 host bits → 2⁸ = 256 addresses (254 usable for hosts; 2 reserved).
- The subnet mask `255.255.255.0` *is* `/24`.

### 4.2 Common CIDR sizes
| CIDR | Subnet mask | Host bits | Total addresses | Usable hosts |
|------|-------------|-----------|-----------------|--------------|
| `/8` | 255.0.0.0 | 24 | 16,777,216 | 16,777,214 |
| `/16` | 255.255.0.0 | 16 | 65,536 | 65,534 |
| `/24` | 255.255.255.0 | 8 | 256 | 254 |
| `/30` | 255.255.255.252 | 2 | 4 | 2 |
| `/32` | 255.255.255.255 | 0 | 1 | 1 (a single host) |

> Rule: **usable hosts = 2^(host bits) − 2** (subtracting the **network address** — all host bits 0 — and the **broadcast address** — all host bits 1).

### 4.3 Reserved addresses in a subnet
For `192.168.1.0/24`:
- **Network address:** `192.168.1.0` (identifies the subnet itself).
- **Broadcast address:** `192.168.1.255` (sends to all hosts in the subnet).
- **Usable host range:** `192.168.1.1` – `192.168.1.254`.

### 4.4 Why subnetting matters
- **Organization:** separate departments/services into logical networks.
- **Security:** isolate subnets (e.g., put databases in a private subnet — firewalls control crossing).
- **Efficiency:** allocate appropriately-sized blocks instead of wasting addresses.
- **Routing:** routers forward between subnets; smaller broadcast domains reduce noise.

---

## 5. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "My IP is what websites see" | Your **private** IP differs from your **public** (NAT) IP |
| Confusing subnet mask with IP | The mask defines the network/host split, not an address |
| Forgetting the −2 for usable hosts | Network + broadcast addresses aren't host-assignable |
| "IPv6 is just longer IPv4" | Different format, no NAT, auto-config, huge space |
| `localhost` ≠ a real network address | `127.0.0.1`/`::1` is the loopback (this machine) |
| `/24` is "small" | `/24` = 256 addresses; bigger number after `/` = **smaller** network |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **Binding servers:** `0.0.0.0` (all interfaces) vs `127.0.0.1` (local only) matters for Spring Boot/Docker (`server.address`) — bind to `0.0.0.0` in containers (Phase 10).
- **Cloud networking (Phase 16.7):** VPCs, subnets, security groups, and private/public subnets are built on CIDR — databases in private subnets, app servers in public ones.
- **Kubernetes (Phase 10.2):** pod/service networking and CIDR ranges underpin cluster networking.
- **Security (Phase 15):** IP allow/deny lists, rate limiting by IP, and network segmentation rely on understanding addresses/subnets.
- **Debugging:** distinguishing localhost vs LAN vs public IPs is essential when services can't reach each other.

---

## 7. Quick Self-Check Questions

1. How many bits is an IPv4 address? An IPv6 address? How many addresses each?
2. What does a subnet mask do, and how does it relate to CIDR `/n`?
3. What are the three private IPv4 ranges, and what is NAT for?
4. Why is your laptop's IP different from the IP a website sees?
5. For `/24`, how many total and usable host addresses are there, and why the difference?
6. What are the network and broadcast addresses of `192.168.1.0/24`?
7. What does `127.0.0.1` / `::1` mean? What about `0.0.0.0`?
8. How do you abbreviate an IPv6 address?

---

## 8. Key Terms Glossary

- **IP address:** logical L3 address identifying a device on a network.
- **IPv4 / IPv6:** 32-bit / 128-bit addressing schemes.
- **Octet:** one 8-bit segment of an IPv4 address.
- **Subnet mask:** marks the network vs host portion of an address.
- **Network / host portion:** which network / which device.
- **CIDR (`/n`):** notation giving the number of network bits.
- **Subnetting:** dividing a network into smaller subnetworks.
- **Private/public IP:** non-routable internal vs internet-routable address.
- **NAT:** translating many private IPs to one public IP.
- **Loopback (`127.0.0.1`/`::1`):** the local machine itself.
- **Network/broadcast address:** subnet identifier / all-hosts address (not host-usable).

---

*Previous topic: **OSI & TCP/IP Models**.*
*Next topic: **DNS (Domain Name System)**.*
