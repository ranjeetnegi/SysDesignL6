# System Design Interview Preparation — Senior SWE Roadmap

A condensed study path through the [main repo](README.md) for **Senior SWE** (Google L5 / Senior Engineer) system design interview prep.

---

## Target & Goal

**Target:** Google L5 / Senior Engineer system design interviews.

**Goal:** Ship a clear, scalable design in 45 minutes — requirements, capacity estimate, main components, and key trade-offs.

---

## Section 1 — Mindset (skim)

Skim for interview presence: designing under ambiguity and leading the conversation.

| Chapter | Topic | Link |
|---------|--------|------|
| 4 | Staff Engineer Mindset — Designing Under Ambiguity | [Chapter 4](Section1/Chapter_4.md) |
| 6 | Communication and Interview Leadership | [Chapter 6](Section1/Chapter_6.md) |

*Optional:* Ch 1–3, 5 for full mindset context.

---

## Section 2 — System Design Framework

| Chapter | Topic | Link |
|---------|--------|------|
| 7 | System Design Framework | [Chapter 7](Section2/Chapter_7.md) |
| 8 | Phase 1 — Users & Use Cases | [Chapter 8](Section2/Chapter_8.md) |
| 9 | Phase 2 — Functional Requirements | [Chapter 9](Section2/Chapter_9.md) |
| 10 | Phase 3 — Scale & Capacity Planning | [Chapter 10](Section2/Chapter_10.md) |
| 11 | Cost, Efficiency, and Sustainable Design | [Chapter 11](Section2/Chapter_11.md) |
| 12 | Phase 4 & Phase 5 — NFRs, Assumptions, Constraints | [Chapter 12](Section2/Chapter_12.md) |
| 13 | End-to-End System Design Using the 5-Phase Framework | [Chapter 13](Section2/Chapter_13.md) |

---

## Section 3 — Distributed Systems (core)

| Chapter | Topic | Link |
|---------|--------|------|
| 14 | Consistency Models — Guarantees, Trade-offs, and Failure Behavior | [Chapter 14](Section3/Chapter_14.md) |
| 15 | Replication and Sharding — Scaling Without Losing Control | [Chapter 15](Section3/Chapter_15.md) |
| 17 | Backpressure, Retries, and Idempotency | [Chapter 17](Section3/Chapter_17.md) |
| 18 | Queues, Logs, and Streams — Choosing the Right Asynchronous Model | [Chapter 18](Section3/Chapter_18.md) |
| 19 | Failure Models and Partial Failures — Designing for Reality | [Chapter 19](Section3/Chapter_19.md) |

*Optional for depth:* Ch 16 (Leader Election & Locks), Ch 20 (CAP Theorem).

---

## Section 4 — Data Systems (required reference)

Use these when practicing problems that need DB, cache, or async flows. Don’t skip — L5 interviews expect you to justify storage and caching choices.

| Chapter | Topic | Use when practicing | Link |
|---------|--------|----------------------|------|
| 21 | Databases — Choosing, Using, and Evolving Data Stores | Storage, search, auth, any system with persistent data | [Chapter 21](Section4/Chapter_21.md) |
| 22 | Caching at Scale — Redis, CDN, and Edge | Rate limiter, URL shortener, feed, read-heavy systems | [Chapter 22](Section4/Chapter_22.md) |
| 23 | Event-Driven Architectures — Kafka, Streams | Notifications, feed, async processing, fan-out | [Chapter 23](Section4/Chapter_23.md) |

---

## Section 5 — Senior-Level Design Problems

Practice end-to-end with these 13 problems. Each has a full walkthrough: requirements, scale, architecture, failure handling.

**Priority:** Do **6–8 problems in depth** (timed, out loud, with diagrams). Quality beats covering all 13. Start with the must-practice set below.

### Must practice (high frequency at L5)

| # | Chapter | Link |
|---|---------|------|
| 29 | Single-Region Rate Limiter | [Chapter 29](Section5/Chapter_29.md) |
| 39 | Real-Time Chat | [Chapter 39](Section5/Chapter_39.md) |
| 28 | URL Shortener | [Chapter 28](Section5/Chapter_28.md) |
| 30 | Distributed Cache (Single Cluster) | [Chapter 30](Section5/Chapter_30.md) |
| 32 | Notification System | [Chapter 32](Section5/Chapter_32.md) |
| 31 | Object / File Storage System | [Chapter 31](Section5/Chapter_31.md) |

### Second tier (strong to have)

| # | Chapter | Link |
|---|---------|------|
| 33 | Authentication System (AuthN) | [Chapter 33](Section5/Chapter_33.md) |
| 34 | Search System | [Chapter 34](Section5/Chapter_34.md) |
| 38 | API Gateway | [Chapter 38](Section5/Chapter_38.md) |
| 36 | Background Job Queue | [Chapter 36](Section5/Chapter_36.md) |
| 35 | Metrics Collection System | [Chapter 35](Section5/Chapter_35.md) |
| 37 | Payment Flow | [Chapter 37](Section5/Chapter_37.md) |
| 40 | Configuration Management | [Chapter 40](Section5/Chapter_40.md) |

---

## Practice strategy

- **Timebox:** 45 minutes end-to-end. Allocate roughly: clarify & requirements (5–8 min), scale/capacity (5 min), high-level design (15–20 min), deep dive 1–2 components (10–15 min), trade-offs & wrap-up (5 min).
- **Practice out loud:** Talk through your reasoning. Say your assumptions. Draw the diagram as you go. Either practice with a partner or record yourself and replay.
- **Use the framework:** Follow the 5-phase flow (Section 2) every time so it becomes automatic.
- **Reference as needed:** When a problem needs DB choice, caching, or events, open the relevant Section 4 chapter and Section 3 (e.g. sharding, consistency) so your answers are precise.

---

## Roadmap Summary

```
Section 1          Skim Ch 4, 6 — mindset & communication
Section 2          ✅ Full (Ch 7–13) — Framework
Section 3          ✅ Ch 14, 15, 17, 18, 19 — Distributed systems core
Section 4          ✅ Ch 21, 22, 23 — Required reference (use when practicing)
Section 5          ✅ 13 problems; prioritize 6–8 in depth (must-practice set first)
Section 6          Skip for Senior
```
