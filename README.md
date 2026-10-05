# SALESTORM 2026 — High-Scale E-Commerce Flash Sale System Architecture

> **Team Name**: SALESTORM  
> **Repository**: [https://github.com/rupeshsah86/SALESTORM_SD_Hackathon_26.git](https://github.com/rupeshsah86/SALESTORM_SD_Hackathon_26.git)  
> **Challenge**: 10,000 concurrent customers competing for 100 inventory units under extreme spike load with zero oversell, sub-second latency, idempotent payments, and fault-tolerant order fulfillment.

---

## Executive Summary & System Overview

SALESTORM 2026 is an enterprise-grade, design-first distributed architecture built specifically to solve the extreme concurrency and reliability challenges of high-demand e-commerce flash sales. 

### The Flash Sale Dilemma
When 10,000 buyers hit "Buy Now" in the exact same millisecond for a stock of only 100 items:
1. **Traditional RDBMS locking** (e.g., `SELECT ... FOR UPDATE`) collapses under lock contention and connection pool exhaustion.
2. **Naive caching** causes race conditions resulting in overselling (selling 150 items when only 100 exist).
3. **Network timeouts & retry storms** lead to duplicate payment charges and inconsistent order states.

### SALESTORM Architectural Solution
SALESTORM solves this with a **Three-Tier Concurrency & Reliability Barrier**:
- **Layer 1: Edge Rate Limiting & Virtual Waiting Room (Token Bucket / Redis Leaky Bucket)**: Sheds excess load cleanly at the API Gateway level.
- **Layer 2: Two-Phase Atomic Inventory Reservation (Redis Lua Script + DB Optimistic Locking)**: Atomic inventory decrements in Redis execute in single-threaded $O(1)$ time, issuing an encrypted, time-bounded **Reservation Token (10-minute TTL)**. The underlying PostgreSQL database uses **Optimistic Concurrency Control (OCC)** via version vectors.
- **Layer 3: Saga Orchestration & Transactional Outbox Pattern**: Decouples payment processing and fulfillment using asynchronous event-driven state machines with automatic saga compensations, dead-letter queues (DLQ), and idempotent payment handling.

---

## Key System Design Highlights

| Architectural Dimension | Architectural Mechanism | Guarantee / Metric |
| :--- | :--- | :--- |
| **Concurrency Control** | Redis Lua Script Atomic Decrement + OCC Versioning | **Zero Oversell** (Exactly 100 items sold) |
| **Reservation TTL** | Redis Key Expiry + Background Cleanup Cron + Delay Queue | 10-minute hold; auto-release on drop |
| **Payment Reliability** | Transactional Inbox/Outbox + Unique Idempotency Hash | **At-most-once** payment execution |
| **Order Resilience** | Orchestrated Saga Pattern with Circuit Breakers & DLQs | Self-healing under 30s downstream outage |
| **Scalability** | Horizontal Partitioning + Gateway Traffic Control | Scalable from 10k to 500k+ concurrent requests |
| **Observability** | Prometheus Metrics + Jaeger Distributed Tracing + Structured JSON | Sub-millisecond bottleneck visibility |

---

## Directory & Documentation Structure

```
SALESTORM_TEAM_NAME/
├── 01_Requirements/
│   └── Requirements_and_Assumptions.md    # FR, NFR, Assumptions, Hard Guarantees vs SLA Targets
├── 02_HLD/
│   ├── System_Context_Diagram.md          # C4 Context (Users, Gateways, External PG, Shipping)
│   ├── Container_Architecture.md          # C4 Container (Services, Caching, Event Bus, DBs)
│   ├── Component_Diagram.md               # Detailed Microservice Component Interaction
│   └── Deployment_Diagram.md              # Cloud Kubernetes (EKS/GKE) Infrastructure Topology
├── 03_LLD/
│   ├── Class_Diagrams.md                  # Class structures for Inventory, Payment, Order Services
│   ├── Sequence_Diagram_Purchase_Reservation.md # Step-by-step reservation workflow & Lua script logic
│   ├── Sequence_Diagram_Payment.md        # Payment authorization, webhook, failure & timeout handling
│   ├── Sequence_Diagram_Order.md          # Order creation, saga orchestration & reconciliation
│   └── State_Diagrams.md                  # State machines for Inventory Reservation & Order Lifecycle
├── 04_Database/
│   └── Database_ER_Design.md              # PostgreSQL Relational Schema, Indexes, Locks, Partitioning
├── 05_API/
│   └── API_Specification.md               # REST / Event API Specs, Request/Response payloads, Errors
├── 06_SOLID/
│   └── SOLID_Mapping.md                   # Concrete application of Single Responsibility, Open/Closed, etc.
├── 07_Design_Patterns/
│   └── Design_Patterns_Mapping.md         # Strategy, Factory, State, Observer, Adapter, Circuit Breaker
├── 08_Scalability_Reliability/
│   └── Scalability_Reliability_Design.md  # Scaling 10k -> 500k, Circuit Breaker, Chaos & 30s Outage Recovery
├── 09_Security_Observability/
│   └── Security_Observability_Design.md   # OAuth2/JWT, Rate Limiting, PCI-DSS, Distributed Tracing & Metrics
├── 10_ADR/
│   └── Architecture_Decision_Records.md   # 8 Comprehensive ADRs (Redis Lua, Saga, DB OCC, Outbox, etc.)
├── 11_AI_Assisted_Validation/
│   └── Simulation_Notes.md                # Concurrency simulation walkthrough & Load test validation notes
├── 12_Presentation/
│   └── Final_5_Minute_Pitch.md            # Jury Presentation Script & Step-by-step 10,000-user walk-through
└── README.md                              # Main Index & Overview
```

---

## Suggested Team Roles & Jury Defense Strategy

### Jury Defense Strategy
1. **Defend Concurrency First**: Demonstrate how Redis Lua script eliminates race conditions at the memory layer before any DB connection is opened.
2. **Defend Financial Integrity**: Explain how the Payment Idempotency Key combined with the Transactional Outbox prevents double-charging even when the network drops mid-transaction.
3. **Defend Resilience**: Show how the Saga Orchestration pattern cleanly rollbacks inventory reservations if payment fails or times out after 10 minutes.

---
*SALESTORM 2026 Architecture Blueprint — Designed for High-Scale Production Systems.*
