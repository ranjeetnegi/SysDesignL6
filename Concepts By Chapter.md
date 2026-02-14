# Concepts by Chapter — SysDesignL6 Index

This file maps **concepts** to the **chapters** where they are covered in detail. Use it to find where to read about a topic or to verify coverage.

**Format:** Concept → Chapter(s) (chapter number and short title).

---

## How to Use

- **Find where a concept is covered:** Search for the concept below; the listed chapters contain the main treatment.
- **See what a chapter covers:** Look up the chapter number in the "By Chapter" section at the end.
- **Quick reference:** Concepts are grouped by category; the same concept may appear in multiple chapters.

---

## 1. Fundamentals & Mindset

| Concept | Covered in |
|--------|------------|
| Scalability | 2, 3, 5, 7, 10, 11, 12, 15, 21, 24, 27 |
| Availability | 2, 10, 11, 12, 14, 19, 21, 22, 24 |
| Reliability | 2, 5, 7, 10, 12, 17, 19, 21, 37 |
| Latency vs Throughput vs Bandwidth | 10, 14, 21, 22, 30, 38, 48 |
| Client-Server Architecture | 7, 8, 13, 28, 38, 48 (implicit in all designs) |
| Single Point of Failure | 21, 42 |
| High Availability vs Fault Tolerance | 16, 19, 22, 24 |
| Vertical vs Horizontal Scaling | 10, 21, 28, 36, 43 |
| Trade-off reasoning & decision-making | 1, 4, 5, 6, 12, 14, 19, 20, 21 |
| Scope, impact, ownership (Staff level) | 1, 2, 3, 6 |
| Designing under ambiguity | 4, 6, 7 |
| API as contract / API evolution | 3, 6, 8, 27, 38, 42, 47, 48 |

---

## 2. Data & Storage

| Concept | Covered in |
|--------|------------|
| Databases (selection, trade-offs) | 5, 7, 11, 21, 26 |
| SQL vs NoSQL | 5, 21, 28 |
| ACID | 7, 21, 37, 53, 56 |
| BASE / eventual consistency | 14, 20, 21, 52 |
| Data Replication | 15, 19, 21, 24, 30, 43, 45, 46, 47, 49, 50, 52, 54, 55, 56 |
| Read Replicas | 10, 12, 15, 21, 25, 28, 43 |
| Sharding | 10, 15, 21, 34, 41, 43, 49, 53 |
| Data Partitioning | 10, 15, 21, 33, 34, 41, 43, 45, 49, 50, 53, 54, 55 |
| Consistent Hashing | 15, 30, 47 |
| Partition key design / hot partitions / skew | 10, 15, 21, 41, 43, 49 |
| Denormalization | 21 |
| Indexing (B-trees, secondary indexes) | 21, 34, 49 |
| OLTP vs OLAP (workload → store) | 11, 21, 46, 50 |
| Soft delete, archival, retention | 25, 39, 43, 45, 49, 51, 55, 57 |
| Multi-tenant data isolation | 21, 49, 52, 53 |
| Schema migration / backfill | 21, 27, 40 |
| Object Storage | 31, 55, 57, 21, 46, 49, 51 |
| Block vs File vs Object (storage types) | 31 (object/file), 55, 57 |
| Distributed File Systems | 31, 55 (referenced) |
| Data Compression | 22, 35, 46, 54, 55, 57 |

---

## 3. Caching

| Concept | Covered in |
|--------|------------|
| Caching (general, layers) | 22, 28, 30, 42, 43, 48 |
| Cache Invalidation | 22, 30, 34, 40 |
| Cache Eviction Policies (LRU, LFU, TTL) | 22, 30, 42 |
| Write-through / write-back / cache-aside | 22, 30, 42, 43 |
| Cache stampede / thundering herd | 22, 30, 42, 43 |
| Hot keys / large objects | 10, 15, 22, 30, 41, 42, 43 |
| CDN | 22, 24, 43, 48, 57 |
| Distributed Cache | 22, 30, 42 |
| Cache hit rate / measuring cache effectiveness | 21, 22, 28, 42 |

---

## 4. Load Balancing & Networking

| Concept | Covered in |
|--------|------------|
| Load Balancing | 24, 38, 48, 55 |
| Load Balancing Algorithms (round-robin, least-connections) | 38, 48 |
| Proxy vs Reverse Proxy | 38 |
| DNS / Geo-DNS | 24, 29, 48 |
| DNS Load Balancing | 24, 48 |
| Anycast Routing | 24, 41, 48 |
| HTTP vs HTTPS | 28, 38 |
| TCP vs UDP | 18, 38 |
| TLS/SSL (termination, mTLS) | 22, 33, 38, 47, 52, 55 |

---

## 5. APIs & Security

| Concept | Covered in |
|--------|------------|
| API Design | 2, 6, 8, 48 |
| REST APIs | 6, 8, 45, 48 |
| Authentication vs Authorization | 33, 52 |
| Session-based vs Token-based Authentication | 33, 52 |
| OAuth / OAuth2 | 33, 52 |
| JWT | 33, 38, 52 |
| Rate Limiting | 29, 38, 41, 48, 52 |
| Idempotency (write APIs, events) | 17, 18, 23, 28, 31, 36, 37, 45, 46, 51, 53, 56 |

---

## 6. Consistency & Distributed Theory

| Concept | Covered in |
|--------|------------|
| CAP Theorem | 14, 19, 20 |
| Consistency Models | 14, 16, 22, 12 |
| Eventual vs Strong Consistency | 12, 14, 19, 20, 21 |
| Network Partitions | 19, 20, 21, 24, 27, 39, 46 |
| Split-Brain | 19, 21, 24, 43, 44 |
| Quorum | 14, 16, 19, 20 |
| Consensus Algorithms | 14, 16, 19, 20 |
| Paxos | 11, 12, 14, 16, 19, 20, 26 |
| Raft | 14, 16, 19, 20, 23, 40, 53 |
| Gossip Protocol | 16, 38, 52 |
| Leader Election | 16, 19, 31, 53 |
| Heartbeats | 16, 19, 45, 47, 53 |
| Clock Synchronization | 16 |
| Logical Clocks / Lamport Timestamps / Vector Clocks | 16, 45 |

---

## 7. Transactions & Messaging

| Concept | Covered in |
|--------|------------|
| Distributed Transactions | 15, 21, 56 |
| Two-Phase Commit (2PC) | 15, 16 |
| SAGA Pattern | 15, 23, 56 |
| Outbox Pattern | 18, 21, 23, 40, 51 |
| Delivery Semantics (at-least-once, exactly-once) | 18, 23, 20, 53, 54 |
| Change Data Capture (CDC) | 23, 30, 34, 40 |
| Message Queues | 18, 23, 32, 36, 49, 50, 51 |
| Pub/Sub | 11, 23, 40, 47, 54 |
| Synchronous vs Asynchronous Communication | 15, 18, 19 |
| Queue vs Log vs Stream | 18 |
| Backpressure | 17, 18, 43, 44, 48, 49, 51, 55 |

---

## 8. Architecture Patterns

| Concept | Covered in |
|--------|------------|
| Microservices Architecture | 3, 11, 23, 27, 52 |
| Monolithic Architecture | 3, 27, 52 |
| Serverless Architecture | 11, 23, 53, 55 |
| Event-Driven Architecture | 22, 23, 27 |
| WebSockets | 38, 39, 45, 47, 54, 55 |
| API Gateways | 11, 27, 38, 48 |
| Long Polling | 39, 45, 47, 54 |
| Server-Sent Events (SSE) | 38, 47, 54, 45 |

---

## 9. Design Questions (from your list)

| Question / Concept | Primary chapters |
|--------------------|------------------|
| When to split monolith into services | 3, 27 |
| Request flow end-to-end | 28, 36, 38, 48 |
| Handle 10× traffic | 10, 28, 30, 32, 33, 36 |
| Vertical vs horizontal scaling | 10, 21, 28, 36 |
| Sync vs async | 15, 18, 19 |
| Queue vs direct RPC | 18 |
| Backpressure design | 17, 18, 43, 44, 48, 49 |
| Protect downstream (rate limiting) | 29, 41, 48 |
| API evolution without breaking clients | 3, 47, 48, 42 |
| Idempotency for write APIs | 17, 31, 36, 45, 51 |
| OLTP vs OLAP modeling | 11, 21 |
| SQL vs NoSQL vs search | 21, 34, 49 |
| Sharding strategy | 15, 21, 43, 49 |
| Partition key / hot partitions | 10, 15, 21, 43 |
| Indexing read-heavy vs write-heavy | 21 |
| Soft delete, archival, retention | 25, 39, 43, 55 |
| Multi-tenant isolation | 21, 49, 52 |
| Migration / backfill | 21, 27 |
| Where to cache (layers) | 22, 42, 48 |
| Write-through / write-back / write-around | 22, 30 |
| Cache invalidation | 22, 30 |
| Cache stampede / thundering herd | 22, 30, 42 |
| Hot keys, large objects | 22, 30, 42 |
| Measure cache effectiveness | 22, 42 |
| Pagination (cursor, offset) | 49 |
| Rate limiting (don’t break good users) | 29, 38, 41, 48 |
| Eventual vs strict consistency | 12, 14, 19, 20, 21 |
| Consistency across services | 14, 16, 22 |
| Distributed transactions without 2PC (SAGA) | 15, 23, 56 |
| Outbox / inbox for reliable events | 18, 23, 40 |

---

## By Chapter (What Each Chapter Covers)

Quick lookup: which concepts are covered in each chapter.

| Ch | Title | Main concepts covered |
|----|--------|------------------------|
| 1 | How Google Evaluates Staff Engineers | Evaluation dimensions, L5 vs L6, trade-off reasoning |
| 2 | Scope, Impact, and Ownership | Scalability, availability, scope, ownership |
| 3 | Designing Systems That Scale Across Teams | When to split monolith, API as contract, platform vs service |
| 4 | Staff Engineer Mindset — Designing Under Ambiguity | Ambiguity, assumptions, scoping |
| 5 | Trade-offs, Constraints, and Decision-Making | Trade-offs, SQL/NoSQL, ACID, scaling stages |
| 6 | Communication and Interview Leadership | API design, structure, REST |
| 7 | System Design Framework | Reliability, ACID, framework phases |
| 8 | Phase 1 — Users & Use Cases | API design, user types, use cases |
| 9 | Phase 2 — Functional Requirements | Functional requirements |
| 10 | Phase 3 — Scale: Capacity Planning and Growth | Scaling, hot keys, partitioning, peak, vertical/horizontal |
| 11 | Cost, Efficiency, and Sustainable Design | OLTP, cost, Raft/Paxos, microservices, serverless |
| 12 | Phase 4 & Phase 5 — NFRs, Assumptions, Constraints | NFRs, consistency, replication, Paxos/Raft |
| 13 | End-to-End System Design (5-Phase Framework) | Full framework application |
| 14 | Consistency Models | CAP, consistency models, quorum, Paxos/Raft, replication |
| 15 | Replication and Sharding | Replication, read replicas, sharding, partitioning, consistent hashing, 2PC, SAGA, hot partitions |
| 16 | Leader Election, Coordination, Distributed Locks | Leader election, Raft, Paxos, quorum, heartbeats, logical/vector clocks, 2PC, clock sync |
| 17 | Backpressure, Retries, and Idempotency | Backpressure, retries, idempotency |
| 18 | Queues, Logs, and Streams | Sync vs async, queue vs log vs stream, delivery semantics, outbox pattern |
| 19 | Failure Models and Partial Failures | Network partitions, split-brain, leader election, heartbeats, Raft/Paxos, consistency |
| 20 | CAP Theorem (Applied Case Studies) | CAP, eventual vs strong, network partitions, Raft/Paxos |
| 21 | Databases | SQL/NoSQL, ACID, vertical/horizontal scaling, replication, sharding, indexing, denormalization, migration, outbox, split-brain, multi-tenant |
| 22 | Caching at Scale | Cache invalidation, TTL, stampede, thundering herd, CDN, TLS, consistency for caching, write-through/cache-aside |
| 23 | Event-Driven Architectures | Event-driven, Kafka, delivery semantics, outbox, SAGA, CDC, Pub/Sub |
| 24 | Multi-Region Systems | Geo-replication, DNS, Anycast, load balancing, split-brain, availability |
| 25 | Data Locality, Compliance, System Evolution | Retention, deletion, GDPR-style, compliance |
| 26 | Cost, Efficiency, Sustainable Design | Database cost, Paxos, cost modeling |
| 27 | System Evolution, Migration, and Risk Management | Schema migration, monolith to microservices, backfill, rollback |
| 28 | URL Shortener | Request flow, scaling 10×, vertical/horizontal, cache, idempotency |
| 29 | Single-Region Rate Limiter | Rate limiting, protecting downstream, overload protection |
| 30 | Distributed Cache (Single Cluster) | Distributed cache, stampede, invalidation, consistent hashing, 10× scale |
| 31 | Object / File Storage System | Object storage, idempotency, durability |
| 32 | Notification System | Message queue, scaling 10×, delivery |
| 33 | Authentication System | AuthN, session vs token, OAuth, JWT, scaling |
| 34 | Search System | Search, indexing, CDC, partitioning |
| 35 | Metrics Collection System | Metrics, compression, push vs pull |
| 36 | Background Job Queue | Request flow, idempotency, scaling, vertical/horizontal |
| 37 | Payment Flow | ACID, idempotency, reliability |
| 38 | API Gateway | Request pipeline, TLS, rate limiting, proxy, WebSockets, SSE, JWT, gossip |
| 39 | Real-Time Chat | Hot/warm/cold data, soft delete, WebSocket, long polling |
| 40 | Configuration Management | Outbox, Raft, push vs poll |
| 41 | Global Rate Limiter | Rate limiting, partitioning, Anycast, hot keys |
| 42 | Distributed Cache | Cache layers, stampede, schema evolution, hit rate |
| 43 | News Feed | Partitioning, retention, backpressure, load shedding, consistency |
| 44 | Real-Time Collaboration | Backpressure, load shedding, Lamport/vector clocks |
| 45 | Messaging Platform | WebSockets, partitioning, idempotency, Lamport timestamps |
| 46 | Metrics / Observability System | TSDB, compression, idempotency, multi-tenant |
| 47 | Configuration, Feature Flags & Secrets | Pub/Sub, SSE, long-poll, TLS, versioning |
| 48 | API Gateway / Edge Request Routing | Request flow, rate limiting, backpressure, load shedding, API evolution |
| 49 | Search / Indexing System | Partitioning, sharding, multi-tenant, backpressure, pagination |
| 50 | Recommendation / Ranking System | Data modeling, partitioning, events |
| 51 | Notification Delivery System | Fan-out, idempotency, outbox, queues |
| 52 | Authentication & Authorization | AuthN/AuthZ, session vs token, mTLS, OAuth, JWT, multi-tenant |
| 53 | Distributed Scheduler / Job Orchestration | Heartbeats, Raft/Paxos, partitioning, at-least-once |
| 54 | Feature Experimentation / A/B Testing | Assignment, partitioning, SSE, long-poll, events |
| 55 | Log Aggregation & Query System | Tiered storage, compression, object storage, backpressure |
| 56 | Payment / Transaction Processing | SAGA, ACID, distributed transactions |
| 57 | Media Upload & Processing Pipeline | Object storage, tiering, WebSockets, compression |

---

## Notes

- **Primary** = chapter has a dedicated section or deep treatment.
- **Mentioned** = concept appears in examples, tables, or trade-off discussion.
- Section 1–2: mindset and framework; Section 3–4: distributed systems and data; Section 5–6: applied design problems where concepts are used in context.

*Generated as a reference for SysDesignL6. Update as you add or change chapters.*
