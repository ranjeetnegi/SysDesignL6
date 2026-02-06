# Chapter 37: Payment Flow — Review Report

## Review Method: Sr_MASTER_REVIWER.md (13-Step Process)

---

## STEP 1: Gap Audit Summary

The original chapter was very strong across most dimensions. Gaps identified:

### Failure handling gaps:
- No processor inbound webhook handling (chargebacks, disputes, async authorization expiry notifications from processor)
- No explicit orphaned authorization detection + cleanup flow (auth void rate alert existed, but no operational procedure)

### Ownership & on-call gaps:
- No dedicated "2 AM on-call, 30-minute mitigation window" pressure scenario with the 4 mandatory questions

### Data model & consistency gaps:
- No handling of processor-initiated state changes (chargeback flips "captured" → "disputed" — not in original state machine)

---

## STEP 2: Enrichment Applied

### Gap 1: Ownership Under Pressure (30-min on-call scenarios)
- **Added:** Exercise D0 — "2 AM On-Call: Ledger Imbalance Alert" with all 4 mandatory questions (what to check first, what to avoid, escalation criteria, communication)
- **Added:** Exercise D0b — "2 AM On-Call: Double-Charge Alert" with same structure
- **Location:** Part 18, new Section D (Ownership Under Pressure), before Correctness exercises

### Gap 2: Orphaned Authorization Detection & Cleanup
- **Added:** New subsection in Part 10 — hourly scheduled job detecting authorizations > 6 days old, handling logic (capture, void, or alert), authorization_expired terminal state, customer notification flow
- **L5 relevance:** A Senior engineer doesn't let stale holds sit on customer cards for a week. Proactive cleanup demonstrates operational ownership.

### Gap 3: Processor Inbound Webhooks (Chargebacks & Async Events)
- **Added:** New subsection in Part 10 — Chargeback/dispute handling flow, dispute resolution (won/lost), webhook endpoint design with signature verification, idempotency, out-of-order handling
- **L5 relevance:** Processor-initiated state changes (disputes) move money. Ignoring inbound webhooks means the ledger drifts from financial reality.

### Supporting Updates:
- **State machine** (Part 7): Added `authorization_expired`, `disputed`, `chargebacked` states with valid transitions
- **Schema** (Part 8): Updated status CHECK constraint to include new states
- **Final Verification** (end of chapter): Updated to reflect all new content

---

## STEP 3: Failure-Awareness Enforcement — VERIFIED

| Required Content | Status |
|------------------|--------|
| Partial failure behavior | ✅ Processor timeout, capture+ledger failure, webhook failure, DB failover |
| Timeout and retry behavior | ✅ 5s/10s timeouts, 3-5 retries, exponential backoff + jitter, check-before-retry |
| Realistic production failure scenario | ✅ Ambiguous timeout double-charge (340 customers, $17K, 7-step response) |

No additions needed.

---

## STEP 4: Rollout, Rollback & Operational Safety — VERIFIED

| Required Content | Status |
|------------------|--------|
| Deployment strategy | ✅ Rolling with canary |
| Canary criteria | ✅ Auth success rate, ledger balance, latency, DLQ |
| Rollout stages | ✅ 1% → 10% → 50% → 100% with bake times |
| Rollback trigger | ✅ Ledger imbalance, double-charge, error rate |
| Bad deployment scenario | ✅ Missing refund ledger entries |

No additions needed.

---

## STEP 5: Senior-Level Judgment — VERIFIED

All major design decisions justified with alternatives and reasoning:
- PostgreSQL vs DynamoDB vs event store
- Synchronous vs async replication
- Auth+capture vs single charge
- Two-layer idempotency design
- No caching for payment data (stale financial data = dangerous)

No additions needed.

---

## STEP 6: Scale Reality — VERIFIED

| Required Content | Status |
|------------------|--------|
| Scale estimates table | ✅ 1×-50× with breakpoints |
| Most fragile assumption | ✅ Single processor, reconciliation job duration |
| Back-of-envelope math | ✅ 100K orders, 50 writes/sec, 100 GB/year |

No additions needed.

---

## STEP 7: Cost & Operability — VERIFIED

| Required Content | Status |
|------------------|--------|
| Cost analysis | ✅ $500/month infra vs $5.25M/month processor fees |
| What NOT to build | ✅ Don't optimize $500 infrastructure; negotiate processor rates |
| Cost vs operability table | ✅ Sync replication, audit table, reconciliation, multi-processor |

No additions needed. The chapter's insight ("infrastructure cost is noise compared to processor fees") is one of the strongest in the series.

---

## STEP 8: Misleading Signals — VERIFIED

| Required Content | Status |
|------------------|--------|
| False confidence table | ✅ Auth success rate, capture rate, refund count, revenue, error rate |
| Actual signals | ✅ Stuck payments, auth without capture, ledger balance, settlement delta |
| Applied to system | ✅ Applied directly to payment flow (not generic) |

No additions needed.

---

## STEP 9: Real-World Application — VERIFIED

| Required Content | Status |
|------------------|--------|
| Rushed decision scenario | ✅ Launch without reconciliation, circuit breaker, automated refund |
| Technical debt + paydown plan | ✅ Week 2/4/8 paydown schedule with cost of carrying debt |

No additions needed.

---

## STEP 10: Diagrams — VERIFIED

| Diagram | Status |
|---------|--------|
| Architecture diagram | ✅ Full component layout with async paths |
| Payment state machine | ✅ All states and transitions visualized |

No additions needed.

---

## STEP 11: Interview Calibration — VERIFIED

| Required Content | Status |
|------------------|--------|
| Evaluator signal table | ✅ 5 signals with payment-specific assessment |
| Strong L5 phrases | ✅ 6 phrases, all payment-domain-specific |
| L4 mistake template | ✅ 4 mistakes (no idempotency, FLOAT, no ledger, timeout=failure) |
| Borderline L5 mistake | ✅ 2 mistakes (no reconciliation, no ledger fix procedure) |
| What distinguishes L5 | ✅ Strong signals section with 6 detailed examples |

No additions needed.

---

## STEP 12: Final Verification

### A. Assessment Statement:

> "This chapter now **meets** Google Senior Software Engineer (L5) expectations."

### B. Checklist of Senior-Level Signals Covered:

| Signal | Status |
|--------|--------|
| End-to-end clarity | ✅ |
| Trade-off justification | ✅ |
| Failure handling | ✅ |
| Scale reasoning | ✅ |
| Cost awareness | ✅ |
| Ownership mindset | ✅ |
| Rollout/rollback | ✅ |
| Interview calibration | ✅ |

### C. Unavoidable Gaps:

- None. All identified gaps were addressed by direct content insertion.

---

## STEP 13: Brainstorming — VERIFIED

All mandatory sections present:
- A. Scale & Load (flash sale, failure order, vertical vs horizontal)
- B. Failure Injection (6 scenarios with full analysis)
- C. Cost & Trade-offs (biggest driver, 30% cut, downtime cost)
- D. Ownership Under Pressure (NEW: 2 AM ledger imbalance + double-charge scenarios)
- E. Correctness & Data Integrity (idempotency, ledger detection, partial failure)
- F. Incremental Evolution (multi-currency, second processor, schema migration)
- G. Deployment & Rollout Safety (stages, bad code, rushed decision)
- H. Interview-Oriented (clarifying questions, non-goals, scope pushback)

---

## Summary of Changes Made

| Change | Location | Type |
|--------|----------|------|
| 2 AM on-call: Ledger imbalance scenario | Part 18 Section D (new) | New content |
| 2 AM on-call: Double-charge scenario | Part 18 Section D (new) | New content |
| Orphaned authorization detection & cleanup | Part 10 (new subsection) | New content |
| Processor inbound webhooks (chargebacks) | Part 10 (new subsection) | New content |
| State machine: disputed, chargebacked, auth_expired | Part 7 | Extended |
| Schema: new status values in CHECK constraint | Part 8 | Extended |
| Final Verification checklist | End of chapter | Updated |
