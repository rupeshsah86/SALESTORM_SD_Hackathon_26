# 12 Final 5-Minute Pitch & Jury Walkthrough

## Overview
This document contains the complete **5-Minute Presentation Script** and **Jury Defense Walkthrough** for presenting the **SALESTORM 2026** architecture at the hackathon.

---

## 1. 5-Minute Presentation Pitch Script

### Minute 0:00 - 1:00 | The Problem & The Challenge
> *"Good morning, respected judges and technical jury.  
> Imagine a high-demand flash sale: **10,000 eager customers** hit 'Buy Now' in the exact same millisecond. But there are only **100 units** available.  
> Traditional e-commerce architectures fail catastrophically in this scenario. RDBMS row locks collapse under lock contention, database connection pools exhaust, race conditions lead to overselling 150 units, and gateway timeouts result in customers being charged twice without getting an order.  
> Today, we present **SALESTORM 2026** — an enterprise-grade distributed system design that guarantees **zero oversell, sub-50ms reservation latency, at-most-once payment idempotency, and self-healing fault tolerance**."*

---

### Minute 1:00 - 2:00 | Three-Tier Architecture & Zero Oversell
> *"To solve extreme concurrency, SALESTORM introduces a **Three-Tier Traffic & Concurrency Barrier**:  
> First, at the edge, Cloudflare Edge WAF and Kong API Gateway enforce Token Bucket rate limiting and a Virtual Waiting Room to shed excessive traffic safely.  
> Second, for inventory reservation, we eliminate database locking entirely. Our Flash Inventory Microservice executes a single-threaded, atomic **Redis Lua Script** in memory ($O(1)$ time). In just 4 milliseconds, Redis decrements the stock counter from 100 to 0 and issues a cryptographically signed **10-Minute Reservation Token**.  
> The 101st customer receives an immediate HTTP 409 'Sold Out' response without hitting our primary database. We guarantee **exactly 100 sales — zero oversell, zero database lock contention**."*

---

### Minute 2:00 - 3:00 | Financial Integrity & Payment Idempotency
> *"What happens when a user submits payment?  
> Network timeouts and panicked double-clicks are the #1 cause of duplicate charges. SALESTORM enforces an **Idempotency Key Ledger** combined with the **Transactional Outbox Pattern**.  
> When a payment arrives, we check our Redis/Postgres idempotency ledger. Duplicate requests return the original transaction result in under 1ms without calling the payment gateway twice.  
> When payment succeeds, we write the payment record and the outbound event to an `outbox` table within the **same local database transaction**, eliminating dual-write inconsistencies."*

---

### Minute 3:00 - 4:00 | Resilience & 30-Second Outage Self-Healing
> *"Now let's talk about real-world failure.  
> Suppose a customer's payment succeeds, but the **Order Service crashes for 30 seconds** due to a cloud node failure.  
> Does the customer lose their money? No!  
> Payment success events are stored in **Apache Kafka persistent event logs** with a 7-day retention window. When Kubernetes restarts the Order Service containers after 30 seconds, the service reconnects to Kafka, resumes from its uncommitted offset, reads the `payment.authorized` event, and creates the order record via Optimistic Concurrency Control.  
> **Zero data loss. Zero dropped orders. Complete self-healing recovery.**"*

---

### Minute 4:00 - 5:00 | Summary & Jury Q&A Walkthrough
> *"To summarize: SALESTORM combines **Redis Lua Atomic Decrements**, **PostgreSQL Optimistic Concurrency Control**, **Saga Orchestration**, and **Transactional Outbox Event Streaming** into a fully defendable, production-ready system design.  
> Thank you, and we are now ready for your questions!"*

---

## 2. Jury Walkthrough: Defending Key Technical Scenarios

### Jury Question 1: "What if the primary Redis node crashes right in the middle of the flash sale?"
- **Answer**: *"We run Redis ElastiCache in **Cluster Mode with Multi-AZ Replication and AOF (Append-Only File) durability**. If a primary Redis node fails, Redis Sentinel automatically promotes a read replica in another Availability Zone within $< 5\text{ seconds}$. Furthermore, our persistent PostgreSQL `inventory` table serves as the source of truth, updated asynchronously via the Transactional Outbox relay."*

### Jury Question 2: "What if 10 users reserve items but abandon their carts for 10 minutes?"
- **Answer**: *"Every reservation token carries an explicit **600-second TTL** in Redis. When the TTL expires, Redis emits a `__keyevent@0__:expired` keyspace notification. Our Expiry Listener worker catches the event, executes `INCR stock` in Redis to restore stock units, and emits an `inventory.released` event to Kafka. Users waiting in the Virtual Waiting Room are notified via WebSockets to claim the freed units."*

### Jury Question 3: "How do you defend against payment gateway HTTP 504 timeouts?"
- **Answer**: *"We wrap third-party payment calls with **Resilience4j Circuit Breakers**. If the payment gateway times out (HTTP 504), we do not assume failure. We return HTTP 202 Accepted (`PENDING_RECONCILIATION`) to the user. Our system resolves the state asynchronously via Payment Webhooks or background polling reconciliation crons."*

---
*Document Version: 1.0.0 — SALESTORM 2026 Pitch & Jury Walkthrough*
