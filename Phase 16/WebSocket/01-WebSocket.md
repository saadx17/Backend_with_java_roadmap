# WebSocket (Real-Time Communication)

> **Phase 16 — Advanced Topics → 16.3 WebSocket**
> Goal: Understand real-time bidirectional communication — WebSocket vs HTTP/polling/SSE, the protocol & handshake, STOMP, Spring WebSocket support, and scaling considerations.

---

## 0. The Big Picture

Plain HTTP (Phase 0.2/7) is **request-response**: the client asks, the server answers, connection closes. That's poor for **real-time** features (chat, live dashboards, notifications, multiplayer) where the **server needs to push** data to the client. **WebSocket** provides a **persistent, full-duplex (bidirectional)** connection over a single TCP socket — both sides can send anytime.

```
   HTTP:       client ──req──► server ──resp──► (closed). Server can't push later.
   WebSocket:  client ◄════════════════════► server   (one persistent open pipe, both push freely)
```

> Builds on HTTP/TCP (Phase 0.2). An alternative to polling and SSE for server push. Integrates via Spring WebSocket/STOMP; can carry GraphQL subscriptions (16.2) and reactive streams (16.1 — hot publishers); has distinct scaling concerns (13.3, 12).

---

## 1. Real-Time Options Compared ⭐

| Technique | Direction | How | Use for |
|-----------|-----------|-----|---------|
| **Short polling** | Client pulls | Client requests every N seconds | Simple, low-freq updates (wasteful) |
| **Long polling** | Client pulls (held) | Server holds the request until data, then responds | Better than short polling; legacy fallback |
| **SSE (Server-Sent Events)** ⭐ | **Server → client** (one-way) | Server streams over a long-lived HTTP response (`text/event-stream`) | Server push only (notifications, feeds, dashboards) — simple, auto-reconnect, over HTTP |
| **WebSocket** ⭐ | **Bidirectional** | Persistent full-duplex socket | Two-way real-time (chat, gaming, collaborative editing) |
```
   SSE:        server ════► client   (one-way push, simple, plain HTTP)
   WebSocket:  server ◄═══► client   (two-way, more powerful, more complex)
```
> ⭐ **Choose the simplest that fits:** if you only need **server→client** push (live prices, notifications), **SSE** is simpler (plain HTTP, auto-reconnect, works through most proxies, easy with WebFlux `Flux` — 16.1). Use **WebSocket** when you need **true bidirectional** low-latency communication. ⚠️ Don't reach for WebSocket when SSE or even polling suffices.

---

## 2. The WebSocket Protocol

- Starts as an HTTP request that **upgrades** the connection (`Upgrade: websocket` handshake), then switches to the WebSocket protocol (`ws://` or secure `wss://`).
- After the handshake, it's a **persistent TCP connection** carrying lightweight **frames** (text/binary) in both directions — no HTTP headers per message (low overhead).
```
   Client → GET /ws  Upgrade: websocket, Connection: Upgrade, Sec-WebSocket-Key: ...
   Server → 101 Switching Protocols
   ═══════ now a full-duplex frame-based channel (ws:// or wss://) ═══════
```
| Aspect | Note |
|--------|------|
| **Handshake** | HTTP upgrade → 101 Switching Protocols |
| **`ws://` / `wss://`** | Plain / **TLS-secured** (use wss in production — 15.3) |
| **Frames** | Small text/binary messages; ping/pong keep-alive |
| **Stateful** | The connection persists → ⚠️ scaling/state implications (§4) |
> ⭐ Use **`wss://` (TLS)** in production (15.3) and to traverse proxies/firewalls reliably. The persistent, **stateful** connection is what makes WebSocket powerful — and what complicates horizontal scaling (§4).

---

## 3. STOMP & Spring WebSocket

Raw WebSocket is just a message pipe with no structure. **STOMP** (Simple Text Oriented Messaging Protocol) adds a **messaging layer** on top — destinations, subscriptions, pub/sub semantics (like a lightweight broker — recall messaging 11.1) — which Spring supports first-class.
```java
@Configuration @EnableWebSocketMessageBroker
class WsConfig implements WebSocketMessageBrokerConfigurer {
    public void registerStompEndpoints(StompEndpointRegistry reg) {
        reg.addEndpoint("/ws").withSockJS();        // handshake endpoint (+ SockJS fallback)
    }
    public void configureMessageBroker(MessageBrokerRegistry reg) {
        reg.enableSimpleBroker("/topic");           // in-memory broker for /topic/* (or relay to RabbitMQ — §4)
        reg.setApplicationDestinationPrefixes("/app");
    }
}

@Controller
class ChatController {
    @MessageMapping("/chat")          // client sends to /app/chat
    @SendTo("/topic/messages")        // broadcast to all subscribers of /topic/messages
    ChatMessage handle(ChatMessage msg) { return msg; }
}
```
| Concept | Meaning |
|---------|---------|
| **STOMP destination** | A named channel (`/topic/...`, `/queue/...`) |
| **`@MessageMapping`** | Handle a message sent to a destination |
| **`@SendTo` / `SimpMessagingTemplate`** | Broadcast / push to subscribers programmatically |
| **SockJS** | Fallback (long-polling) when WebSocket isn't available |
> ⭐ **STOMP gives you pub/sub semantics** (topics, subscriptions) over WebSocket — perfect for chat/broadcast. Spring's `SimpMessagingTemplate` lets the server push to specific users/topics from anywhere in your code. Without STOMP you handle raw frames yourself (`WebSocketHandler`).

---

## 4. Scaling WebSockets ⭐

WebSocket's **stateful, long-lived** connections clash with stateless horizontal scaling (13.3) — connections are pinned to a specific server instance.
| Challenge | Solution |
|-----------|----------|
| Connection tied to one instance | Sticky sessions at the LB, **or** a shared broker (below) |
| Broadcasting across instances | ⭐ Use an **external broker relay** (RabbitMQ/Kafka — Phase 11) so a message published on instance A reaches subscribers on instance B |
| Many open connections (memory/file descriptors) | Tune limits; consider reactive (Netty/WebFlux — 16.1) for many connections on few threads |
| Auth on the persistent connection | Authenticate at handshake (token — 15.2); re-check authorization |
| Load balancer support | LB must support WebSocket upgrade & long-lived connections |
```
   instance A ─┐                          ┌─ instance B
   clients ────┼──► [ RabbitMQ/Kafka broker relay ] ◄──┼──── clients
               └─ a message on A is relayed to subscribers on B (cross-instance fan-out)
```
> ⭐ **The key scaling pattern: replace Spring's in-memory broker with an external broker relay** (RabbitMQ STOMP / Kafka — Phase 11) so broadcasts reach clients connected to *any* instance. ⚠️ Reactive WebFlux (16.1) handles **many concurrent connections** efficiently (event loop vs thread-per-connection). Authenticate at the **handshake** (15.2) since the connection is long-lived.

---

## 5. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| WebSocket when SSE/polling suffices | Use the simplest fit (§1) |
| `ws://` in production | Use `wss://` (TLS) (§2, 15.3) |
| In-memory broker → broadcasts miss other instances | External broker relay (RabbitMQ/Kafka) (§4) |
| Ignoring stateful-connection scaling | Sticky sessions or broker relay; reactive for scale (§4) |
| No auth at handshake | Authenticate the connection (token) (§4, 15.2) |
| Thread-per-connection under high load | Reactive (Netty/WebFlux) (§4, 16.1) |
| No keep-alive/heartbeat | Use ping/pong / STOMP heartbeats (§2) |
| LB not configured for WebSocket | Ensure upgrade + long-lived connection support (§4) |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- Builds on **HTTP/TCP** (0.2); secured with **wss/TLS** (15.3) and handshake auth (15.2).
- **Spring WebSocket + STOMP**; **SockJS** fallback; `SimpMessagingTemplate` for server push.
- **Scaling** ↔ stateless design/sticky sessions (13.3), **external broker** (Kafka/RabbitMQ — Phase 11).
- **Reactive** (16.1) for many concurrent connections; carries **GraphQL subscriptions** (16.2).
- **Observability** (9.2/12.7) for connection metrics.
- Optional/advanced; useful for real-time features in **Project 5/7**.

---

## 7. Quick Self-Check Questions

1. Why is plain HTTP poor for real-time server push?
2. Compare short polling, long polling, SSE, and WebSocket — direction and use case of each.
3. When is SSE the better choice over WebSocket?
4. How does the WebSocket handshake/upgrade work, and what are `ws://` vs `wss://`?
5. What does STOMP add over raw WebSocket?
6. How do `@MessageMapping` and `@SendTo` work in Spring?
7. Why are WebSockets hard to scale horizontally?
8. What is a broker relay, and why is it the key scaling pattern?
9. Why authenticate at the handshake, and why prefer reactive for many connections?
10. What is SockJS for?

---

## 8. Key Terms Glossary

- **WebSocket:** persistent full-duplex (bidirectional) connection over TCP.
- **SSE (Server-Sent Events):** one-way server→client streaming over HTTP.
- **Short/long polling:** client-pull techniques (legacy fallbacks).
- **Handshake / upgrade / 101:** HTTP→WebSocket protocol switch.
- **`ws://` / `wss://`:** plain / TLS-secured WebSocket.
- **Frame / ping-pong:** lightweight message / keep-alive.
- **STOMP / destination / `@MessageMapping` / `@SendTo`:** messaging layer & Spring mappings.
- **SockJS:** fallback transport when WebSocket is unavailable.
- **Broker relay:** external broker (RabbitMQ/Kafka) for cross-instance fan-out.
- **Sticky session:** LB pinning a connection to one instance.

---

*This is the note for **Section 16.3 — WebSocket**.*
*Previous section in roadmap: **16.2 GraphQL**.*
*Next section in roadmap: **16.4 Elasticsearch**.*
