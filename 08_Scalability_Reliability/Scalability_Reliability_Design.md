# 08 Scalability & Reliability Design

## Overview
This document defines the high-scale strategy, fault tolerance mechanisms, circuit breakers, retry strategies, and disaster-recovery protocols that allow **SALESTORM 2026** to scale seamlessly from **10,000 to 500,000 concurrent users** while remaining resilient against downstream service outages.

---

## 1. Scaling Strategy: 10,000 to 500,000 Concurrent Requests

```
  500,000 RPS (Extreme Surge Load)
         │
         ▼
 ┌────────────────────────────────────────────────────────┐
 │   Layer 1: Cloudflare Edge WAF & Virtual Waiting Room  │  --> Sheds 98% of passive traffic
 └────────────────────────────────────────────────────────┘      (Users view queue position: #14,291)
         │ (Admits 10,000 active buyers/sec)
         ▼
 ┌────────────────────────────────────────────────────────┐
 │   Layer 2: Kong API Gateway (Token Bucket Limiter)     │  --> Rate limits 1 req/user/3s
 └────────────────────────────────────────────────────────┘
         │
         ▼
 ┌────────────────────────────────────────────────────────┐
 │   Layer 3: Redis Cluster (Lua Single-Threaded Engine)  │  --> Executes atomic decrements ($< 2\text{ms}$)
 └────────────────────────────────────────────────────────┘      (Returns SOLD_OUT to 9,900 users instantly)
         │
         ▼ (Only 100 successful reservations)
 ┌────────────────────────────────────────────────────────┐
 │   Layer 4: PostgreSQL RDBMS & Kafka Async Saga Bus     │  --> Handles financial updates via OCC
 └────────────────────────────────────────────────────────┘
```

### Key Scaling Drivers
1. **Virtual Waiting Room (Edge Queueing)**: When traffic exceeds 10,000 RPS, Cloudflare Workers intercept incoming HTTP connections and issue a signed Waiting Room Token. Users poll queue status via WebSockets without touching microservices.
2. **Redis Cluster Sharding**: Inventory counters for different SKUs are sharded across separate Redis master nodes, ensuring zero cross-slot lock contention.
3. **Database Connection Pooling (PgBouncer)**: Microservices connect via PgBouncer in `transaction` pooling mode, keeping open database connections under 200 even when pod counts scale to 50+.

---

## 2. Resilience & Fault Tolerance Mechanisms

### 1. Circuit Breaker Configuration (Resilience4j)
- **Sliding Window**: 20 requests.
- **Failure Rate Threshold**: $50\%$.
- **Slow Call Rate Threshold**: $50\%$ (Calls slower than $1,500\text{ms}$).
- **Wait Duration in Open State**: 10,000ms (10 seconds).
- **Behavior**: When open, payment requests immediately trigger the fallback method without attempting HTTP connection, preserving backend CPU.

### 2. Exponential Backoff with Decorrelated Jitter
When calling external APIs or retrying Kafka message processing:
$$T_{\text{wait}} = \min(T_{\text{max}}, \text{random}(T_{\text{base}}, T_{\text{wait}} \times 3))$$
- Prevents the **Thundering Herd Problem** during gateway recovery.

---

## 3. Disaster Recovery Scenario: Payment Success + Order Service 30s Outage

### Failure Walkthrough
1. **t = 0s**: Buyer pays $99.00 successfully. Payment Gateway returns HTTP 200 OK. Payment Service writes to `payments` table and publishes `payment.authorized` event to Kafka.
2. **t = 1s**: **Order Service Pods Crash** (e.g., node evicted by cloud provider).
3. **t = 1s to 30s**: Kafka retains the `payment.authorized` message at Offset `#109482` in partition 2. Customer payment is **100% safe**. No duplicate charges occur.
4. **t = 31s**: Kubernetes Kubelet restarts Order Service pods.
5. **t = 33s**: Order Service reconnects to Kafka, reads Offset `#109482`, verifies reservation status in PostgreSQL, creates the order record, and commits the offset.
6. **Result**: **Zero Data Loss. Zero Customer Impact. Recovery completed within 35 seconds.**

---

## 4. Chaos Engineering Validation Scenarios

| Test Case | Injected Fault | Expected System Behavior | Success Criteria |
| :--- | :--- | :--- | :--- |
| **Chaos Test 1** | Kill 2 out of 3 Redis Master Nodes. | Redis Sentinel / Cluster auto-promotes Replicas within 5 seconds. | Zero oversell; $< 0.1\%$ transient error rate. |
| **Chaos Test 2** | Block Payment Gateway Network (100% packet drop). | Circuit Breaker opens; system switches to Async Webhook Queue. | HTTP 202 returned to users; no thread starvation. |
| **Chaos Test 3** | Simulate 5,000 duplicate payment retry clicks. | Idempotency Ledger intercepts requests in Redis ($< 1\text{ms}$). | Exactly 1 payment captured per reservation. |

---
*Document Version: 1.0.0 — SALESTORM 2026 Scalability Design*
