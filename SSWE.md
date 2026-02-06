# System Design Interview Preparation — Roadmaps

Condensed study paths through the [main repo](README.md) for **Senior SWE** and **Staff SWE** system design prep. Use the roadmap that matches your target level; Senior is a subset and foundation for Staff.

---

# Senior SWE Roadmap

**Target:** Google L5 / Senior Engineer system design interviews.

**Goal:** Ship a clear, scalable design in 45 minutes — requirements, capacity estimate, main components, and key trade-offs.

---

## Phase 1 — Framework (Required)

**Section 2 — System Design Framework**

| Chapter | Topic | Link |
|---------|--------|------|
| 7 | The Staff-Level System Design Framework | [Chapter 7](Section2/Chapter_7.md) |
| 8 | Phase 1 — Users & Use Cases | [Chapter 8](Section2/Chapter_8.md) |
| 9 | Phase 2 — Functional Requirements | [Chapter 9](Section2/Chapter_9.md) |
| 10 | Phase 3 — Scale & Capacity Planning | [Chapter 10](Section2/Chapter_10.md) |
| 11 | Cost, Efficiency, and Sustainable Design | [Chapter 11](Section2/Chapter_11.md) |
| 12 | Phase 4 & Phase 5 — NFRs, Assumptions, Constraints | [Chapter 12](Section2/Chapter_12.md) |
| 13 | End-to-End System Design Using the 5-Phase Framework | [Chapter 13](Section2/Chapter_13.md) |

---

## Phase 2 — Distributed Systems (Required)

**Section 3 — Distributed Systems** (high-signal chapters)

| Chapter | Topic | Link |
|---------|--------|------|
| 14 | Consistency Models — Guarantees, Trade-offs, and Failure Behavior | [Chapter 14](Section3/Chapter_14.md) |
| 17 | Backpressure, Retries, and Idempotency | [Chapter 17](Section3/Chapter_17.md) |
| 18 | Queues, Logs, and Streams — Choosing the Right Asynchronous Model | [Chapter 18](Section3/Chapter_18.md) |
| 19 | Failure Models and Partial Failures — Designing for Reality | [Chapter 19](Section3/Chapter_19.md) |

*Optional for depth:* Ch 15 (Replication & Sharding), Ch 16 (Leader Election & Locks), Ch 20 (CAP Theorem).

---

## Phase 3 — Data Systems (Optional — Skim)

**Section 4 — Data Systems & Global Scale**

Skim section [README](Section4/README.md) and use chapters as reference when a design problem touches databases, caching, event streams, or multi-region. No need to read cover-to-cover for Senior.

| Chapter | Topic | Link |
|---------|--------|------|
| 21 | Databases at Staff Level | [Chapter 21](Section4/Chapter_21.md) |
| 22 | Caching at Scale | [Chapter 22](Section4/Chapter_22.md) |
| 23 | Event-Driven Architectures | [Chapter 23](Section4/Chapter_23.md) |

---

## Phase 4 — Senior Design Problems (Required)

**Section 5 — Senior-Level Design Problems**

Practice end-to-end with these 13 problems. Each has a full walkthrough: requirements, scale, architecture, failure handling.

| # | Chapter | Link |
|---|---------|------|
| 28 | URL Shortener | [Chapter 28](Section5/Chapter_28.md) |
| 29 | Single-Region Rate Limiter | [Chapter 29](Section5/Chapter_29.md) |
| 30 | Distributed Cache (Single Cluster) | [Chapter 30](Section5/Chapter_30.md) |
| 31 | Object / File Storage System | [Chapter 31](Section5/Chapter_31.md) |
| 32 | Notification System | [Chapter 32](Section5/Chapter_32.md) |
| 33 | Authentication System (AuthN) | [Chapter 33](Section5/Chapter_33.md) |
| 34 | Search System | [Chapter 34](Section5/Chapter_34.md) |
| 35 | Metrics Collection System | [Chapter 35](Section5/Chapter_35.md) |
| 36 | Background Job Queue | [Chapter 36](Section5/Chapter_36.md) |
| 37 | Payment Flow | [Chapter 37](Section5/Chapter_37.md) |
| 38 | API Gateway | [Chapter 38](Section5/Chapter_38.md) |
| 39 | Real-Time Chat | [Chapter 39](Section5/Chapter_39.md) |
| 40 | Configuration Management | [Chapter 40](Section5/Chapter_40.md) |

---

## Senior Roadmap Summary

```
Section 1          [Optional — skim for mindset]
Section 2          ✅ Full (Ch 7–13) — Framework
Section 3          ✅ Ch 14, 17, 18, 19 — Distributed systems core
Section 4          [Optional — skim / reference]
Section 5          ✅ Full (Ch 28–40) — 13 design problems
Section 6          [Skip for Senior; use for Staff prep]
```

---

# Staff SWE Roadmap

**Target:** Google L6 / Staff Engineer system design interviews.

**Goal:** Demonstrate scope definition, cross-team thinking, trade-off leadership, and design under ambiguity — not just a correct diagram.

---

## Phase 1 — Mindset & Evaluation (Required)

**Section 1 — Staff Engineer Mindset & Evaluation**

| Chapter | Topic | Link |
|---------|--------|------|
| 1 | How Google Evaluates Staff Engineers in System Design Interviews | [Chapter 1](Section1/Chapter_1.md) |
| 2 | Scope, Impact, and Ownership at Google Staff Engineer Level | [Chapter 2](Section1/Chapter_2.md) |
| 3 | Designing Systems That Scale Across Teams (Staff Perspective) | [Chapter 3](Section1/Chapter_3.md) |
| 4 | Staff Engineer Mindset — Designing Under Ambiguity | [Chapter 4](Section1/Chapter_4.md) |
| 5 | Trade-offs, Constraints, and Decision-Making at Staff Level | [Chapter 5](Section1/Chapter_5.md) |
| 6 | Communication and Interview Leadership for Google Staff Engineers | [Chapter 6](Section1/Chapter_6.md) |

---

## Phase 2 — Framework (Required)

**Section 2 — System Design Framework**

Same as Senior: Ch 7–13. [Section 2 table above](#phase-1--framework-required) applies.

---

## Phase 3 — Distributed Systems (Required)

**Section 3 — Distributed Systems** (full section recommended)

| Chapter | Topic | Link |
|---------|--------|------|
| 14 | Consistency Models | [Chapter 14](Section3/Chapter_14.md) |
| 15 | Replication and Sharding | [Chapter 15](Section3/Chapter_15.md) |
| 16 | Leader Election, Coordination, and Distributed Locks | [Chapter 16](Section3/Chapter_16.md) |
| 17 | Backpressure, Retries, and Idempotency | [Chapter 17](Section3/Chapter_17.md) |
| 18 | Queues, Logs, and Streams | [Chapter 18](Section3/Chapter_18.md) |
| 19 | Failure Models and Partial Failures | [Chapter 19](Section3/Chapter_19.md) |
| 20 | CAP Theorem — Behavior Under Partition | [Chapter 20](Section3/Chapter_20.md) |

---

## Phase 4 — Data Systems & Global Scale (Required)

**Section 4 — Data Systems & Global Scale**

| Chapter | Topic | Link |
|---------|--------|------|
| 21 | Databases at Staff Level | [Chapter 21](Section4/Chapter_21.md) |
| 22 | Caching at Scale | [Chapter 22](Section4/Chapter_22.md) |
| 23 | Event-Driven Architectures | [Chapter 23](Section4/Chapter_23.md) |
| 24 | Multi-Region Systems | [Chapter 24](Section4/Chapter_24.md) |
| 25 | Data Locality, Compliance, and System Evolution | [Chapter 25](Section4/Chapter_25.md) |
| 26 | Cost, Efficiency, and Sustainable System Design | [Chapter 26](Section4/Chapter_26.md) |
| 27 | System Evolution, Migration, and Risk Management | [Chapter 27](Section4/Chapter_27.md) |

---

## Phase 5 — Senior Problems (Warm-up)

**Section 5 — Senior-Level Design Problems**

Do at least 4–6 problems from [Section 5](#phase-4--senior-design-problems-required) to build fluency before Staff-level problems.

---

## Phase 6 — Staff Design Problems (Required)

**Section 6 — Staff-Level Design Problems**

Multi-region, cross-team scope, migration, and organizational trade-offs. Prioritize completed chapters first.

| # | Chapter | Status | Link |
|---|---------|--------|------|
| 41 | Global Rate Limiter | Done | [Chapter 41](Section6/Chapter_41.md) |
| 42 | Distributed Cache | Done | [Chapter 42](Section6/Chapter_42.md) |
| 43 | News Feed | Done | [Chapter 43](Section6/Chapter_43.md) |
| 44 | Real-Time Collaboration | Done | [Chapter 44](Section6/Chapter_44.md) |
| 45 | Messaging Platform | Done | [Chapter 45](Section6/Chapter_45.md) |
| 46 | Metrics / Observability System | Done | [Chapter 46](Section6/Chapter_46.md) |
| 47 | Configuration, Feature Flags & Secrets Management | Done | [Chapter 47](Section6/Chapter_47.md) |
| 48 | API Gateway / Edge Request Routing System | Done | [Chapter 48](Section6/Chapter_48.md) |
| 49–57 | Search, Recommendation, Notifications, Auth, Scheduler, A/B, Logs, Payment, Media | Planned | — |

---

## Staff Roadmap Summary

```
Section 1          ✅ Full (Ch 1–6) — Mindset & evaluation
Section 2          ✅ Full (Ch 7–13) — Framework
Section 3          ✅ Full (Ch 14–20) — Distributed systems
Section 4          ✅ Full (Ch 21–27) — Data systems & global scale
Section 5          ✅ 4–6 problems — Warm-up
Section 6          ✅ All available (Ch 41–48+ as added) — Staff problems
```

---

## Quick Comparison

| | Senior SWE | Staff SWE |
|---|------------|-----------|
| **Section 1** | Optional skim | Required |
| **Section 2** | Required | Required |
| **Section 3** | Core chapters (14, 17, 18, 19) | Full section |
| **Section 4** | Optional / reference | Required |
| **Section 5** | All 13 problems | 4–6 as warm-up |
| **Section 6** | Skip | Required (8+ chapters available) |
| **Focus** | Solid design, capacity, trade-offs | Scope, ambiguity, cross-team, leadership |
