# Elasticsearch (Search & Analytics)

> **Phase 16 — Advanced Topics → 16.4 Elasticsearch**
> Goal: Understand Elasticsearch — what full-text search engines do, the inverted index, core concepts (index/document/mapping/analyzer), querying, when to use it vs a relational DB, keeping it in sync, and Spring Data Elasticsearch.

---

## 0. The Big Picture

Relational databases (Phase 4) are great at structured queries and transactions but **bad at full-text search** — fuzzy matching, relevance ranking, "search across everything" — at scale. **Elasticsearch** is a distributed **search and analytics engine** (built on Apache Lucene) optimized for **fast full-text search**, relevance scoring, and aggregations over huge datasets. It's the **E** in the **ELK/Elastic stack** (recall log aggregation — 12.7).

```
   PostgreSQL: WHERE description LIKE '%laptop%'   → slow full scan, no ranking, no typo tolerance
   Elasticsearch: search "laptp"  → typo-corrected, relevance-ranked, fast results across millions of docs
```

> Complements (doesn't replace) the relational DB (Phase 4). It's the search store in **CQRS read models** (11.4), the log store in **observability** (12.7), and integrates via Spring Data Elasticsearch. Touches indexing concepts (4.4) and data sync/CDC (11.2).

---

## 1. Why a Search Engine? The Inverted Index ⭐

The core data structure that makes full-text search fast is the **inverted index** — instead of "doc → words," it maps "**word → which docs contain it**" (like a book's index).
```
   Documents:  doc1="fast laptop"  doc2="laptop bag"  doc3="fast car"
   Inverted index:
      "fast"   → [doc1, doc3]
      "laptop" → [doc1, doc2]
      "bag"    → [doc2]
   Search "laptop" → instantly [doc1, doc2], ranked by relevance
```
| Search-engine capability | vs RDBMS `LIKE` |
|--------------------------|------------------|
| **Full-text relevance ranking** (TF-IDF/BM25) | None — just match/no-match |
| **Tokenization / stemming** ("running"→"run") | Exact substring only |
| **Fuzzy / typo tolerance** | None |
| **Synonyms, autocomplete, highlighting** | Manual/none |
| **Fast at scale** | `LIKE '%x%'` can't use indexes (4.4) → slow |
| **Aggregations/analytics** | Possible but heavier |
> ⭐ The **inverted index** + analyzers are why search engines beat `LIKE` for text: they pre-process text into terms so lookups are O(index), with ranking and linguistic features built in. (RDBMS full-text extensions exist — Postgres `tsvector` — but Elasticsearch is purpose-built and scales further.)

---

## 2. Core Concepts

| Concept | Meaning | RDBMS analogy (loose) |
|---------|---------|------------------------|
| **Index** | A collection of documents | ~ a table |
| **Document** | A JSON record | ~ a row |
| **Field** | A key in the document | ~ a column |
| **Mapping** | Schema: field types & how they're analyzed | ~ table schema |
| **Analyzer** | Pipeline that tokenizes/normalizes text (lowercase, stemming, stop-words) | (no analog) |
| **Shard** | A partition of an index (Lucene index) — for scale | ~ partition (4.4/13.3) |
| **Replica** | A copy of a shard — for HA + read throughput | ~ read replica (13.3) |
| **Cluster / node** | The distributed system / a server | (distributed DB) |
> ⭐ **Analyzers** are central to relevance: at index time and query time, text is tokenized & normalized (lowercase, remove stop-words, apply stemming/synonyms) so "Laptops" matches "laptop". Choosing/customizing analyzers tunes search quality. **Shards/replicas** make Elasticsearch distributed and scalable (sharding — 13.3).

---

## 3. Querying

Elasticsearch has a rich JSON **Query DSL**:
```jsonc
POST /products/_search
{
  "query": {
    "bool": {
      "must":   [ { "match": { "name": "laptop" } } ],     // full-text, analyzed, ranked
      "filter": [ { "range": { "price": { "lte": 1000 } } } ],  // exact filter, not scored
      "should": [ { "match": { "brand": "dell" } } ]        // boosts relevance if matched
    }
  },
  "aggs": { "by_brand": { "terms": { "field": "brand.keyword" } } }  // analytics/facets
}
```
| Query type | Use |
|-----------|-----|
| **`match`** (full-text) | Analyzed, relevance-scored text search |
| **`term`** (exact) | Exact match on keyword/number (not analyzed) |
| **`bool`** (`must`/`should`/`filter`/`must_not`) | Combine conditions |
| **`range`** | Numeric/date ranges |
| **Aggregations** | Facets, stats, grouping (analytics) |
> ⭐ Key distinction: **`match` (full-text, analyzed, scored)** vs **`term` (exact, keyword)**. Use `match` for search boxes, `term`/`filter` for exact attributes (and `filter` for non-scored conditions = faster + cacheable). **Aggregations** power faceted search and analytics dashboards (Kibana).

---

## 4. When to Use It (and the Sync Problem) ⭐

| Use Elasticsearch for | Keep in the RDBMS for |
|-----------------------|-----------------------|
| Full-text/fuzzy search, autocomplete | Source of truth / transactions (4.5) |
| Relevance-ranked results | Strong consistency, joins, constraints |
| Log/metrics analytics (ELK — 12.7) | Financial/critical writes |
| Faceted search, dashboards | Normalized relational data (4.3) |

> ⚠️ ⭐ **Elasticsearch is NOT your primary database.** It's not built for ACID transactions or as a system of record — it's **near-real-time** and eventually consistent. The standard pattern: **RDBMS = source of truth; Elasticsearch = a derived search index** kept in sync.

### 4.1 Keeping it in sync
```
   PostgreSQL (truth) ──► sync ──► Elasticsearch (search index)
   Options: (a) dual-write from the app (⚠️ dual-write problem — 11.4)
            (b) async via events/Outbox (11.4) → consumer indexes into ES  ⭐
            (c) CDC with Debezium (11.2 §8) → stream DB changes to ES
```
> ⭐ Don't dual-write naively (it diverges — recall the dual-write problem, 11.4 §5). Prefer **event-driven sync** (Outbox/CDC — 11.4/11.2) so the search index updates reliably from committed DB changes. This makes Elasticsearch a **CQRS read model** (11.4 §2).

---

## 5. Spring Data Elasticsearch

```java
@Document(indexName = "products")                  // maps to an ES index
class Product {
    @Id String id;
    @Field(type = FieldType.Text, analyzer = "standard") String name;   // full-text
    @Field(type = FieldType.Keyword) String brand;                       // exact
    @Field(type = FieldType.Double) double price;
}

interface ProductSearchRepository extends ElasticsearchRepository<Product, String> {
    List<Product> findByNameContaining(String text);   // derived query
}
```
- Spring Data Elasticsearch gives repositories + `ElasticsearchOperations`/criteria for complex queries (like Spring Data JPA — 5.4, but for ES).
- ⚠️ Model the **mapping** deliberately (`Text` vs `Keyword`, analyzers) — it determines search behavior.
> ⭐ Use Spring Data Elasticsearch for app integration, but understand the underlying Query DSL (§3) for advanced search/relevance tuning. (Alternative: OpenSearch, the OSS fork.)

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using ES as the primary/transactional DB | RDBMS = truth; ES = derived index (§4) |
| Dual-writing app → DB + ES | Event/CDC-driven sync (Outbox) (§4.1, 11.4) |
| Expecting immediate consistency | ES is near-real-time/eventual (§4) |
| `term` query on analyzed text (no match) | Use `match` for text, `term` on `.keyword` (§3) |
| Wrong mapping/analyzer → bad relevance | Design Text vs Keyword + analyzers (§2/§5) |
| `LIKE '%x%'` in RDBMS for big text search | Use ES (or Postgres FTS for small scale) (§1) |
| Scored queries where filters suffice | Use `filter` (faster, cacheable) (§3) |
| Ignoring shard/replica sizing | Plan shards for scale/HA (§2, 13.3) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- Complements the **RDBMS** (Phase 4) — not a replacement; relates to indexing (4.4), sharding (13.3).
- Kept in sync via **events/Outbox/CDC** (11.4/11.2) → a **CQRS read model** (11.4).
- The **E** in **ELK** for log aggregation (12.7); analytics dashboards (Kibana — 9.2).
- **Spring Data Elasticsearch** mirrors Spring Data JPA (5.4); GDPR erasure must include ES (15.3).
- Optional/advanced; powers search in **Project 5 (E-Commerce)** & **Project 7**.

---

## 8. Quick Self-Check Questions

1. Why are relational DBs poor at full-text search?
2. What is an inverted index, and why does it make search fast?
3. What do analyzers do, and why do they matter for relevance?
4. Map ES concepts (index/document/mapping/shard/replica) to rough RDBMS analogies.
5. `match` vs `term` queries — when use each? What is `filter` good for?
6. Why is Elasticsearch not a primary/transactional database?
7. How do you keep ES in sync with the source-of-truth DB, and why not dual-write?
8. How does ES relate to CQRS read models?
9. How does Spring Data Elasticsearch model documents and queries?
10. Why must mapping (Text vs Keyword) be designed carefully?

---

## 9. Key Terms Glossary

- **Elasticsearch / Lucene:** distributed search engine / its underlying library.
- **Inverted index:** word→documents mapping enabling fast search.
- **Index / document / field / mapping:** collection / record / key / schema.
- **Analyzer / tokenization / stemming:** text-processing pipeline for search.
- **Shard / replica / cluster / node:** partition / copy / system / server.
- **Query DSL / `match` / `term` / `filter` / aggregation:** querying constructs.
- **Relevance / BM25:** scoring of how well a doc matches.
- **Near-real-time / eventual consistency:** ES freshness model.
- **ELK / Kibana / OpenSearch:** the stack / UI / OSS fork.
- **Spring Data Elasticsearch / `@Document`:** Spring integration.

---

*This is the note for **Section 16.4 — Elasticsearch**.*
*Previous section in roadmap: **16.3 WebSocket**.*
*Next section in roadmap: **16.5 GraalVM & Native Image**.*
