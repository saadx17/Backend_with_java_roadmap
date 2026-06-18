# Apache Kafka

> **Phase 11 — Messaging → 11.2 Apache Kafka**
> Goal: Master Kafka — its architecture (brokers/topics/partitions/offsets), producers/consumers/consumer groups, replication, ZooKeeper vs KRaft, key configs, **Spring Kafka** (`KafkaTemplate`, `@KafkaListener`, DLT), Kafka Streams, Kafka Connect + Debezium (CDC), and the Schema Registry with Avro.

---

## 0. The Big Picture

**Apache Kafka** is a distributed **event-streaming platform** — a durable, replayable, append-only **commit log** (the "stream/log" model from 11.1 §3) built for **very high throughput** and **many independent consumers**. Producers append events to **topics**; events are retained and consumers read at their own pace via **offsets**.

```
   Producers ──► [ TOPIC: orders ]  (partitioned, replicated, retained log)
                  ├ Partition 0: [e0 e1 e2 e3 ...]   ◄ append-only
                  ├ Partition 1: [e0 e1 e2 ...]
                  └ Partition 2: [e0 e1 ...]
                       │
        Consumer Group A (own offsets) ── Consumer Group B (own offsets)  ◄ independent readers
```

> Kafka implements messaging fundamentals (11.1 — pub/sub + queue via groups, at-least-once, back-pressure via pull). It's the engine for **event-driven/CQRS/Event Sourcing/Saga/Outbox** (11.4), **microservices** communication (Phase 12.2), and CDC. Integrated with Spring via **Spring Kafka**.

---

## 1. Architecture

| Concept | Meaning |
|---------|---------|
| **Broker** | A Kafka server; a **cluster** is multiple brokers (scale + fault tolerance) |
| **Topic** | A named stream of events (like a table/category) |
| **Partition** ⭐ | A topic is split into partitions — the unit of **parallelism & ordering** |
| **Offset** | A monotonic id of an event **within a partition**; consumers track it |
| **Record** | An event: key, value, timestamp, headers |
| **Producer** | Appends records to a topic (chooses partition, often by key) |
| **Consumer** | Reads records from partitions, tracking offsets |
| **Consumer group** | A set of consumers sharing the work of a topic |

### 1.1 Partitions — ordering & parallelism ⭐
```
Topic "orders" with 3 partitions:
   records with key=customer42  ──hash──► always Partition 1  (per-key ORDER preserved)
   Partition count = max parallelism (one partition → at most one consumer in a group)
```
- **Ordering is guaranteed only *within* a partition**, not across the topic.
- The **record key** decides the partition (`hash(key) % partitions`) → same key → same partition → ordered. (No key → round-robin.)
- More partitions = more parallel consumers = more throughput. ⚠️ But you can't easily reduce partitions, and too many has overhead.
> ⭐ Choosing the **key** is a core design decision: key by `customerId`/`orderId` to keep a given entity's events ordered while still parallelizing across entities.

---

## 2. Consumer Groups ⭐ (queue + pub-sub unified)

```
   Topic (3 partitions)
   ├ P0 ─┐
   ├ P1 ─┼─► Consumer Group "billing" : C1←P0, C2←P1, C3←P2   (each partition to ONE consumer)
   └ P2 ─┘
        └────► Consumer Group "analytics" : reads ALL partitions, OWN offsets (independent copy)
```
| Rule | Detail |
|------|--------|
| Within a group | Each partition is consumed by **exactly one** consumer → competing consumers / load split (point-to-point — 11.1) |
| Across groups | Every group gets **all** messages independently (pub-sub — 11.1) |
| Scaling | Add consumers up to the partition count (extra consumers idle) |
| **Rebalancing** | When consumers join/leave, partitions are redistributed |
| **Offset commit** | Group stores its position (in the `__consumer_offsets` topic) |
> ⭐ This is how Kafka is **both** a queue (scale within a group) **and** pub-sub (many groups). ⚠️ Max useful consumers per group = number of partitions. **Rebalances** pause consumption briefly — minimize churn.

---

## 3. Replication (Durability & Availability)

Each partition is **replicated** across brokers for fault tolerance.
```
Partition 0:  Leader on Broker 1  ─┬─ Follower on Broker 2  (replicas)
                                   └─ Follower on Broker 3
   Producers/consumers talk to the LEADER; followers replicate; if leader dies → a follower is elected
```
| Concept | Meaning |
|---------|---------|
| **Replication factor** | Copies of each partition (e.g., 3) |
| **Leader / follower** | The replica that serves I/O / replicas that copy it |
| **ISR (In-Sync Replicas)** | Replicas caught up with the leader |
| **`acks`** (producer) | `0` (no wait), `1` (leader only), `all`/`-1` (all ISR — safest) |
| **`min.insync.replicas`** | Minimum ISR required to accept a write |
> ⭐ For durability use **`acks=all` + `replication.factor=3` + `min.insync.replicas=2`** — a write survives one broker failure. `acks=0/1` is faster but can lose data. This is Kafka's at-least-once foundation (11.1 §4).

---

## 4. ZooKeeper vs KRaft

| | **ZooKeeper** (legacy) | **KRaft** (modern) ⭐ |
|--|------------------------|----------------------|
| Role | External system storing cluster metadata | Kafka manages its own metadata via Raft consensus |
| Status | Deprecated; removed in Kafka 4.0 | The default/future (KRaft = Kafka Raft) |
| Ops | Two systems to run | One system — simpler, faster, more scalable |
> ⭐ Older Kafka needed **ZooKeeper** for metadata/leader election. **KRaft** (Kafka 3.x+, mandatory in 4.x) replaces it with a built-in Raft-based controller quorum — fewer moving parts. New clusters use KRaft.

---

## 5. Key Configuration

| Setting | Side | Effect |
|---------|------|--------|
| `acks` | producer | Durability vs speed (§3) |
| `retries` / `enable.idempotence=true` | producer | Safe retries without duplicates (idempotent producer) ⭐ |
| `batch.size` / `linger.ms` | producer | Batching for throughput |
| `compression.type` | producer | gzip/snappy/lz4/zstd → less network/disk |
| `group.id` | consumer | Which consumer group |
| `auto.offset.reset` | consumer | `earliest`/`latest` when no committed offset |
| `enable.auto.commit` | consumer | ⚠️ auto-commit can lose/duplicate; prefer manual commit after processing (11.1 §5) |
| `max.poll.records` | consumer | Batch size per poll (back-pressure — 11.1 §7) |
| `retention.ms` / `retention.bytes` | topic | How long/much to retain (replay window) |
| `cleanup.policy` | topic | `delete` (time/size) or **`compact`** (keep latest per key) |
> ⭐ **Log compaction** (`cleanup.policy=compact`) keeps only the latest value per key — perfect for changelog/state topics (Event Sourcing, CDC — §8/11.4). **Idempotent producer** (`enable.idempotence=true`) prevents duplicate writes on retry.

---

## 6. Spring Kafka

### 6.1 Producing — `KafkaTemplate`
```java
@Service
@RequiredArgsConstructor
class OrderEventPublisher {
    private final KafkaTemplate<String, OrderEvent> kafka;   // injected (Phase 5.1)

    void publish(OrderEvent e) {
        kafka.send("orders", e.orderId(), e);   // topic, KEY (→ partition), value
    }
}
```
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all
      properties: { enable.idempotence: true }
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: billing
      auto-offset-reset: earliest
      enable-auto-commit: false        # manual ack (11.1 §5)
```

### 6.2 Consuming — `@KafkaListener`
```java
@Component
class OrderEventConsumer {
    @KafkaListener(topics = "orders", groupId = "billing")
    void onOrder(OrderEvent e, Acknowledgment ack) {     // manual ack mode
        process(e);          // ⭐ make this IDEMPOTENT (11.1 §4) — at-least-once → duplicates
        ack.acknowledge();   // commit offset AFTER success
    }
}
```

### 6.3 Error handling & DLT (Dead-Letter Topic)
```java
// Retry then send to a dead-letter topic "orders.DLT" (DLQ — 11.1 §6)
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<?,?> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);   // → orders.DLT
    return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));  // 3 retries, 1s apart
}
```
> ⭐ Spring Kafka's **`DeadLetterPublishingRecoverer`** auto-publishes failed records to a `*.DLT` topic after the configured retries — the Kafka realization of the DLQ pattern (11.1 §6). Always run consumers **idempotently** with **manual acks**.

---

## 7. Kafka Streams (Stream Processing)

**Kafka Streams** is a Java library for **transforming/aggregating** streams in real time — read topic(s), process (map/filter/join/window/aggregate), write to topic(s) — with state stores and exactly-once processing.
```java
StreamsBuilder b = new StreamsBuilder();
b.stream("orders", Consumed.with(Serdes.String(), orderSerde))
 .filter((k, o) -> o.total() > 100)
 .groupByKey()
 .count()                                   // stateful aggregation (backed by a state store)
 .toStream()
 .to("big-order-counts");
```
| Kafka Streams concept | Note |
|-----------------------|------|
| **KStream / KTable** | Record stream / changelog table (latest per key — compaction §5) |
| **Stateful ops** | `count`, `aggregate`, **windowing**, **joins** (backed by RocksDB state stores) |
| **Exactly-once (EOS)** | Read-process-write atomically within Kafka |
| Deployment | A regular Java/Spring app (no separate cluster) — scales via partitions |
> ⭐ Use Kafka Streams for **real-time processing/analytics/materialized views** (CQRS read models — 11.4) without a separate processing cluster. (Alternative: Apache Flink for heavier needs.)

---

## 8. Kafka Connect + Debezium (CDC)

**Kafka Connect** is a framework of pluggable **connectors** to stream data **in/out** of Kafka without custom code (source connectors → Kafka; sink connectors → DB/search/etc.).

**Debezium** is a set of **CDC (Change Data Capture)** source connectors: it tails a database's **transaction log** (Postgres WAL/MySQL binlog — Phase 4) and emits each row change as a Kafka event.
```
   PostgreSQL ──(WAL/binlog)──► Debezium (Connect source) ──► Kafka topic "db.public.orders"
                                                                  │
                                       sink connectors ──► Elasticsearch / another DB / cache
```
> ⭐ **CDC via Debezium** is the backbone of the **Outbox pattern** and reliable data propagation across microservices (11.4/12.6): instead of dual-writing to DB + Kafka (which can diverge), you write to the DB, and Debezium reliably turns committed changes into events. Connect handles offsets, retries, and scaling for you.

---

## 9. Schema Registry & Avro

When many services share topics, message **schemas must stay compatible** as they evolve (recall API versioning — Phase 7.1 §8). The **Schema Registry** (Confluent) stores versioned schemas and enforces **compatibility**.
| Concept | Note |
|---------|------|
| **Avro** (or Protobuf/JSON Schema) | Compact **binary** serialization with an explicit schema |
| **Schema Registry** | Central store of schema versions; producers/consumers fetch by id |
| **Compatibility modes** | BACKWARD / FORWARD / FULL — control how schemas may evolve |
| **Benefit** | Small payloads + enforced contracts + safe evolution |
```
Producer serializes with Avro → registers/looks up schema (id) in Registry → only the id travels with the message
Consumer reads the id → fetches the schema → deserializes  (no giant JSON; contract enforced)
```
> ⭐ **Avro + Schema Registry** gives you compact messages and **schema evolution guarantees** — a producer can't ship a breaking change that would crash consumers (the registry rejects incompatible schemas). This is API-contract discipline (7.2) for event streams.

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Expecting topic-wide ordering | Ordering is per-**partition**; key appropriately (§1.1) |
| More consumers than partitions | Extra consumers idle; size partitions for parallelism (§2) |
| Auto-commit offsets → loss/dupes | Manual commit after processing + idempotent consumer (§6, 11.1) |
| `acks=0/1` for critical data | `acks=all` + RF=3 + minISR=2 (§3) |
| No DLT for poison messages | `DeadLetterPublishingRecoverer` + retries (§6, 11.1) |
| Dual-writing DB + Kafka (divergence) | Outbox + Debezium CDC (§8, 11.4) |
| Breaking event schema | Schema Registry + compatibility (§9) |
| Treating Kafka like a queue (delete after read) | It's a retained log; consumers replay via offsets (11.1 §3) |
| Running ZooKeeper for new clusters | Use KRaft (§4) |
| Ignoring rebalance pauses | Minimize consumer churn; tune session timeouts (§2) |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- Implements messaging fundamentals (11.1) — log model, consumer groups, at-least-once, pull back-pressure.
- **Spring Kafka** (`KafkaTemplate`, `@KafkaListener`, DLT) integrates with Spring beans/config (Phase 5.1/5.2); auto-metrics via Micrometer (9.2).
- Powers **async patterns** (11.4): event-driven, CQRS read models (Kafka Streams), **Event Sourcing** (compacted topics), **Saga choreography**, **Outbox + Debezium**.
- **Microservices** (Phase 12.2) — async inter-service communication; **observability** tracing across topics (12.7).
- **CDC/Debezium** reads DB logs (Phase 4); ties to migrations discipline (Phase 8).
- **Idempotency** (Phase 7.1) on consumers; **Redis** dedup (Phase 4.8); **Inbox** (11.4).
- Runs on **Kubernetes** as a **StatefulSet** (10.2); deployed via CI/CD (10.3).
- Used in **Project 7 (Microservices E-Commerce)**.

---

## 12. Quick Self-Check Questions

1. What kind of system is Kafka, and how does the log/offset model differ from a queue (11.1)?
2. Define broker, topic, partition, offset. Why are partitions the unit of ordering & parallelism?
3. How does the record key determine partitioning and ordering?
4. How do consumer groups give you both queue and pub-sub behavior? What limits parallelism?
5. Explain replication: leader/follower, ISR, `acks`, `min.insync.replicas` — and the durable config.
6. ZooKeeper vs KRaft — what changed and why?
7. Show producing with `KafkaTemplate` and consuming with `@KafkaListener` + manual ack + DLT.
8. What is Kafka Streams for, and what are KStream vs KTable?
9. What do Kafka Connect and Debezium do, and how do they enable the Outbox pattern?
10. Why use a Schema Registry + Avro, and what does compatibility enforcement prevent?

---

## 13. Key Terms Glossary

- **Kafka:** distributed event-streaming platform (durable, replayable log).
- **Broker / cluster:** Kafka server / group of brokers.
- **Topic / partition / offset:** stream / parallel-ordered shard / position cursor.
- **Record key:** decides partition (ordering per key).
- **Consumer group / rebalance:** shared consumers / partition redistribution.
- **Replication factor / leader / follower / ISR:** durability mechanics.
- **`acks` / `min.insync.replicas`:** write durability settings.
- **KRaft / ZooKeeper:** modern self-managed metadata / legacy external.
- **Log compaction:** keep latest value per key.
- **Spring Kafka / `KafkaTemplate` / `@KafkaListener` / DLT:** Spring integration & dead-letter topic.
- **Kafka Streams / KStream / KTable:** stream-processing library.
- **Kafka Connect / Debezium / CDC:** connector framework / change-data-capture connectors.
- **Schema Registry / Avro / compatibility:** schema store / binary format / evolution rules.

---

*This is the note for **Section 11.2 — Apache Kafka**.*
*Previous section in roadmap: **11.1 Messaging Fundamentals**.*
*Next section in roadmap: **11.3 RabbitMQ**.*
