# RabbitMQ

> **Phase 11 — Messaging → 11.3 RabbitMQ**
> Goal: Understand RabbitMQ — the AMQP model (exchanges, queues, bindings, routing keys), exchange types, Spring AMQP integration, dead-letter exchanges (DLX), and a clear RabbitMQ vs Kafka comparison.

---

## 0. The Big Picture

**RabbitMQ** is a mature, general-purpose **message broker** built on **AMQP** (Advanced Message Queuing Protocol). Where Kafka (11.2) is a *log/stream*, RabbitMQ is a *smart router of messages into queues* (the "queue" model from 11.1 §3) with rich, flexible **routing**. Producers publish to an **exchange**; the exchange routes copies to **queues** based on **bindings**; consumers drain the queues.

```
   Producer ──► [ EXCHANGE ] ──(bindings + routing key)──► [ QUEUE A ] ──► Consumer A
                     │                                     [ QUEUE B ] ──► Consumer B
                     └─ routing logic lives in the broker (smart broker, dumb consumer)
```

> RabbitMQ implements messaging fundamentals (11.1 — point-to-point, pub/sub, acks, DLQ, prefetch back-pressure). It's integrated via **Spring AMQP** and is the go-to for **task queues, RPC, and complex routing** in microservices (Phase 12.2). Contrast with Kafka (11.2) at the end.

### ⭐ Philosophy: smart broker, dumb consumer
RabbitMQ puts the **intelligence in the broker** (routing, retries, priorities, TTL). Kafka is the opposite — **dumb broker (just a log), smart consumer** (tracks offsets, replays). This shapes when to use each (§5).

---

## 1. The AMQP Model

| Component | Role |
|-----------|------|
| **Producer** | Publishes a message — always to an **exchange** (never directly to a queue) |
| **Exchange** | Receives messages and **routes** them to queues per type + bindings |
| **Queue** | Buffer that holds messages until a consumer acks them |
| **Binding** | A rule linking an exchange to a queue (often with a **routing key** / pattern) |
| **Routing key** | A label on the message the exchange uses to decide routing |
| **Consumer** | Subscribes to a queue and processes messages |
| **Connection / Channel** | TCP connection / lightweight virtual connection (multiplexed) |
| **Virtual host (vhost)** | Logical isolation/namespace within a broker |

```
   publish(exchange="orders", routingKey="order.created", msg)
        │
   [ EXCHANGE "orders" ] ──binding key "order.*"──► [ QUEUE notify ] ──► email service
                         └─binding key "order.created"─► [ QUEUE inventory ] ──► inventory service
```
> ⚠️ Producers **never** publish straight to a queue — always to an **exchange**, which decides routing. A message with no matching binding is **dropped** (unless an alternate exchange is set).

---

## 2. Exchange Types ⭐

| Exchange type | Routing logic | Models |
|---------------|---------------|--------|
| **Direct** | Route to queues whose binding key **exactly equals** the routing key | Point-to-point / by category (11.1) |
| **Topic** | Route by **pattern** matching (`*` = one word, `#` = zero+ words) | Flexible pub-sub (`order.*`, `*.error`) |
| **Fanout** | Route to **all** bound queues, ignoring routing key | Pure broadcast / pub-sub (11.1) |
| **Headers** | Route by **message header** attributes (not routing key) | Complex attribute-based routing |
| **(Default)** | A direct exchange where routing key = queue name | Quick "publish to a queue" |

```
DIRECT:  routingKey "payment" ──► queue bound with "payment"        (exact match)
TOPIC:   routingKey "order.created.eu" ──► queue bound "order.#"     (pattern)
FANOUT:  any message ──► EVERY bound queue                           (broadcast)
```
> ⭐ **Topic exchanges** are the most versatile — wildcard routing (`order.*`, `#.error`) lets you slice events many ways. **Fanout** = simple broadcast (pub-sub). **Direct** = exact-match work routing. This routing flexibility is RabbitMQ's signature strength over Kafka.

---

## 3. Spring AMQP

Spring AMQP (`spring-boot-starter-amqp`, built on `spring-rabbit`) gives `RabbitTemplate` (send) and `@RabbitListener` (receive), plus declarative topology.

### 3.1 Declaring topology (exchanges/queues/bindings)
```java
@Configuration
class RabbitConfig {
    @Bean TopicExchange ordersExchange() { return new TopicExchange("orders"); }
    @Bean Queue inventoryQueue() {
        return QueueBuilder.durable("inventory")
            .withArgument("x-dead-letter-exchange", "orders.dlx")   // DLX (§4)
            .build();
    }
    @Bean Binding b(Queue inventoryQueue, TopicExchange ordersExchange) {
        return BindingBuilder.bind(inventoryQueue).to(ordersExchange).with("order.created");
    }
}
```

### 3.2 Producing — `RabbitTemplate`
```java
@Service @RequiredArgsConstructor
class OrderPublisher {
    private final RabbitTemplate rabbit;
    void publish(OrderEvent e) {
        rabbit.convertAndSend("orders", "order.created", e);   // exchange, routingKey, payload
    }
}
```

### 3.3 Consuming — `@RabbitListener`
```java
@Component
class InventoryConsumer {
    @RabbitListener(queues = "inventory")
    void onOrder(OrderEvent e) {   // auto-ack by default; configure MANUAL for reliability (11.1 §5)
        process(e);                // ⭐ idempotent (at-least-once → duplicates on redelivery)
    }
}
```
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    listener:
      simple:
        acknowledge-mode: manual      # ack after processing (11.1 §5)
        prefetch: 10                  # max unacked in flight = back-pressure (11.1 §7)
        retry: { enabled: true, max-attempts: 3 }
```
> ⭐ **Prefetch** (QoS) limits how many unacked messages a consumer holds → natural **back-pressure** (11.1 §7) and fair dispatch across competing consumers. Use **manual ack** + **idempotent** handlers for reliability.

---

## 4. Dead-Letter Exchange (DLX)

RabbitMQ's realization of the **DLQ pattern** (11.1 §6). A queue can route failed/expired/rejected messages to a **dead-letter exchange**, which routes them to a dead-letter queue.
```
   [ inventory queue ] --(nack after retries / TTL expired / rejected)--► [ orders.dlx ] ──► [ inventory.dlq ]
                                                                                                  │
                                                                          inspect + alert (9.2) + fix + replay
```
A message is dead-lettered when:
- It is **rejected/nacked** with `requeue=false` (after retries), **or**
- Its **TTL** expires, **or**
- The queue **length limit** is exceeded.
```java
QueueBuilder.durable("inventory")
    .withArgument("x-dead-letter-exchange", "orders.dlx")
    .withArgument("x-message-ttl", 60000)          // optional TTL
    .build();
```
> ⭐ Combine DLX with **retry + backoff** (Spring Retry / a delayed-retry queue). RabbitMQ also supports a **delayed-message exchange plugin** and a "retry queue with TTL → DLX back to main" pattern for delayed retries. Always design a DLX for production.

### 4.1 Other RabbitMQ features
| Feature | Note |
|---------|------|
| **Durable queues + persistent messages** | Survive broker restart (set both!) |
| **Publisher confirms** | Broker acks the producer that a message was safely received (reliability) |
| **TTL** | Per-message / per-queue expiry |
| **Priority queues** | Higher-priority messages first |
| **Quorum queues** | Replicated, Raft-based queues for HA (modern replacement for mirrored queues) |
| **Lazy queues** | Keep messages on disk (large backlogs) |
| **Management UI / `rabbitmqctl`** | Web UI + CLI for ops |

---

## 5. RabbitMQ vs Kafka ⭐

| Dimension | **RabbitMQ** (queue/broker) | **Kafka** (log/stream — 11.2) |
|-----------|-----------------------------|-------------------------------|
| Model | Smart broker routes to queues | Dumb broker = append-only log; smart consumers |
| Message lifecycle | **Removed after ack** | **Retained** (replay via offsets) |
| Replay / re-read | ❌ No (gone once consumed) | ✅ Yes (rewind offset) |
| Routing | ⭐ Rich (direct/topic/fanout/headers) | Simple (topic + partition by key) |
| Ordering | Per-queue | Per-partition |
| Throughput | High | **Very high** (sequential disk) |
| Multiple independent consumers | Needs separate queues/bindings | ✅ Native (consumer groups) |
| Latency | Very low | Low |
| Best for | **Task queues, RPC, complex routing, per-message TTL/priority** | **Event streaming, replay, high-volume, many consumers, analytics, CDC** |
| Retention | Transient (a "to-do list") | Durable log (history/replay) |

```
Choose RabbitMQ  →  "route this task to the right worker, with priorities/retries/TTL"
Choose Kafka     →  "durable stream of events many services replay & process at scale"
```
> ⭐ **It's not either/or** — many systems use **both**: RabbitMQ for command/task routing & RPC, Kafka for the event backbone/streaming. Pick per use case: **need replay / many consumers / huge volume → Kafka; need flexible routing / task distribution / per-message control → RabbitMQ.** (Decision mirrors queue-vs-stream — 11.1 §3.)

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Publishing directly to a queue | Publish to an **exchange**; bind queues (§1) |
| Auto-ack then crash → message loss | Manual ack after processing (§3, 11.1 §5) |
| Non-durable queue/messages → lost on restart | Durable queue + persistent messages + publisher confirms (§4.1) |
| No DLX → poison message redelivered forever | Configure DLX + retries (§4, 11.1 §6) |
| No prefetch limit → one consumer hogs/overloads | Set `prefetch` (QoS) (§3) |
| Expecting Kafka-style replay | RabbitMQ removes consumed messages (§5) |
| Wrong exchange type for routing need | Direct/topic/fanout/headers per need (§2) |
| Non-idempotent consumers | Idempotency (at-least-once → duplicates) (11.1 §4) |
| Mirrored queues (deprecated) for HA | Use quorum queues (§4.1) |
| Using RabbitMQ for huge event streams/analytics | Use Kafka for that (§5) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- Implements messaging fundamentals (11.1) — pub/sub & point-to-point, acks, DLQ (DLX), prefetch back-pressure.
- **Spring AMQP** (`RabbitTemplate`, `@RabbitListener`) integrates with Spring beans/config (Phase 5.1/5.2); Micrometer metrics (9.2).
- Used in **async patterns** (11.4) — commands/tasks, Saga (often via routing), event notifications.
- **Microservices** (Phase 12.2) — async/RPC communication, complex routing; **resilience** (Phase 12.3 retries).
- **Idempotency** (Phase 7.1) on consumers; **Redis** dedup (Phase 4.8); **Inbox** (11.4).
- Runs on **Kubernetes** (StatefulSet — 10.2); deployed via CI/CD (10.3).
- Pairs with/contrasts **Kafka** (11.2); used in **Project 5/7** where complex routing/task queues fit.

---

## 8. Quick Self-Check Questions

1. What is AMQP's model — producer → exchange → binding → queue → consumer? Why never publish straight to a queue?
2. Describe the four exchange types and when to use each.
3. What's RabbitMQ's "smart broker, dumb consumer" philosophy vs Kafka's opposite?
4. Show producing with `RabbitTemplate` and consuming with `@RabbitListener`.
5. What is prefetch (QoS), and how does it give back-pressure and fair dispatch?
6. How does a Dead-Letter Exchange work, and what conditions dead-letter a message?
7. What makes a message/queue survive a broker restart (durability)?
8. Give five factors that push you toward Kafka vs RabbitMQ.
9. Why must RabbitMQ consumers usually be idempotent?
10. When might a system use both RabbitMQ and Kafka?

---

## 9. Key Terms Glossary

- **RabbitMQ / AMQP:** message broker / its protocol.
- **Exchange / queue / binding / routing key:** router / buffer / link rule / routing label.
- **Direct / topic / fanout / headers exchange:** exact / pattern / broadcast / attribute routing.
- **Connection / channel / vhost:** TCP / multiplexed virtual connection / namespace.
- **Spring AMQP / `RabbitTemplate` / `@RabbitListener`:** Spring integration.
- **Prefetch (QoS):** max unacked messages per consumer (back-pressure).
- **Publisher confirms:** broker acks safe receipt to the producer.
- **Durable queue / persistent message:** survive restart.
- **DLX (dead-letter exchange):** routes failed/expired/rejected messages (DLQ).
- **TTL / priority / quorum / lazy queues:** expiry / ordering by priority / replicated HA / disk-backed.
- **Smart broker vs smart consumer:** RabbitMQ vs Kafka design philosophy.

---

*This is the note for **Section 11.3 — RabbitMQ**.*
*Previous section in roadmap: **11.2 Apache Kafka**.*
*Next section in roadmap: **11.4 Async Patterns**.*
