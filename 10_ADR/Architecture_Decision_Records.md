# 10 Architecture Decision Records (ADR)

## Overview
This document contains 8 formal **Architecture Decision Records (ADR)** formatted according to the Michael Nygard MADR template. Every major architectural decision is justified with context, decision rationale, trade-offs, and considered alternatives.

---

## Index of Architecture Decision Records

- **ADR-001**: Single-Threaded Redis Lua Script Engine for Inventory Reservation
- **ADR-002**: Orchestrated Saga Pattern for Distributed Checkout Lifecycle
- **ADR-003**: PostgreSQL Optimistic Concurrency Control (OCC) over Pessimistic Locks
- **ADR-004**: Transactional Outbox Pattern for Dual-Write Consistency
- **ADR-005**: Idempotency Key Ledger for At-Most-Once Payment Execution
- **ADR-006**: Apache Kafka for Asynchronous Event Streaming & Buffering
- **ADR-007**: Edge Virtual Waiting Room & Token Bucket Traffic Control
- **ADR-008**: Strategy & Factory Patterns for Multi-Vendor Payment Gateways

---

## ADR-001: Single-Threaded Redis Lua Script Engine for Inventory Reservation

### Context
When 10,000 customers hit the "Buy Now" endpoint simultaneously for 100 inventory units, traditional database transactions suffer from lock contention, deadlock spikes, and connection pool exhaustion.

### Decision
We will execute inventory check, decrement, and token generation in a **single-threaded Redis Lua script** (`EVALSHA`).

### Why This Choice?
Redis executes Lua scripts atomically on its single-threaded event loop. No two requests can interleave or execute concurrently inside the Lua script execution block, providing an $O(1)$ memory-speed guarantee against race conditions.

### Trade-Offs
- **Pros**: Zero oversell guarantee, sub-5ms response time, handles 50,000+ RPS per shard.
- **Cons**: Requires keeping Redis in-memory data synchronized with PostgreSQL persistent storage.

---

## ADR-002: Orchestrated Saga Pattern for Distributed Order Checkout

### Context
The checkout lifecycle spans Inventory, Payment, and Order services. Two-Phase Commit (2PC) is unsuitable due to high network latency and blocking locks across microservices.

### Decision
We choose an **Orchestrated Saga Pattern** managed by `OrderSagaCoordinator`.

### Why This Choice?
Orchestration centralizes state machine visibility, simplifies debugging, and guarantees automated compensation calls (e.g., releasing inventory if payment fails) without circular event dependencies.

### Trade-Offs
- **Pros**: Eventual consistency without blocking database locks; clear state auditability.
- **Cons**: Requires writing explicit compensation logic for every saga step.

---

## ADR-003: PostgreSQL Optimistic Concurrency Control (OCC)

### Context
Persistent database order records must prevent phantom updates when multiple worker pods access the same order record simultaneously.

### Decision
We implement **Optimistic Concurrency Control (OCC)** using integer `version` columns:
`UPDATE orders SET status = 'PAID', version = version + 1 WHERE order_id = $1 AND version = $2`.

### Why This Choice?
Pessimistic locking (`SELECT FOR UPDATE`) holds database locks across network boundaries, causing connection pool exhaustion under load. OCC assumes low collision at the DB layer because Redis has already filtered the traffic down to 100 winners.

### Trade-Offs
- **Pros**: Maximum database read/write throughput; zero blocking row locks.
- **Cons**: Retries are required if an optimistic concurrency update fails due to version mismatch.

---

## ADR-004: Transactional Outbox Pattern for Dual-Write Consistency

### Context
A microservice updating PostgreSQL and publishing a Kafka message can experience partial failures (e.g., DB commits, but network drops before Kafka ACK).

### Decision
We implement the **Transactional Outbox Pattern**. Business state updates and outbound event logs are written to an `outbox_events` table in the **same RDBMS transaction**. A background CDC relayer reads the outbox table and streams events to Kafka.

### Why This Choice?
Guarantees **at-least-once** event publishing without dual-write inconsistencies.

---

## ADR-005: Idempotency Key Ledger for Payment Execution

### Context
Network timeouts (HTTP 504) or user retry double-clicks can result in duplicate payment authorization requests to external payment gateways.

### Decision
Enforce a mandatory `X-Idempotency-Key` header on all payment requests. Store idempotency keys in Redis ($< 1\text{ms}$ check) backed by a PostgreSQL unique index.

### Why This Choice?
Guarantees **at-most-once** payment execution regardless of client retries or network drops.

---

## ADR-006: Apache Kafka for Asynchronous Event Streaming

### Context
High-volume order updates, notifications, and fulfillment tasks must be decoupled from the synchronous HTTP user purchase request path.

### Decision
Use **Apache Kafka** as the persistent message log.

### Why This Choice?
Kafka provides high-throughput disk persistence, consumer group scaling, and replayable event logs with 7-day retention.

---

## ADR-007: Edge Virtual Waiting Room & Token Bucket Traffic Control

### Context
Surge spikes reaching 500,000 RPS can crash edge load balancers before traffic reaches microservice containers.

### Decision
Deploy Cloudflare Edge Virtual Waiting Room coupled with Kong API Gateway Token Bucket rate limiting.

### Why This Choice?
Sheds non-essential traffic at the network edge, admitting only controlled batches of users to the reservation engine.

---

## ADR-008: Strategy & Factory Design Patterns for Payment Gateways

### Context
The platform must support multiple third-party payment gateways (Stripe, Razorpay) dynamically based on geographic region and failure fallback.

### Decision
Implement `IPaymentGatewayStrategy` interfaces instantiated via `PaymentGatewayFactory`.

### Why This Choice?
Adheres to the Open/Closed Principle (OCP). New payment providers can be introduced without modifying core order checkout code.

---
*Document Version: 1.0.0 — SALESTORM 2026 ADR Collection*
