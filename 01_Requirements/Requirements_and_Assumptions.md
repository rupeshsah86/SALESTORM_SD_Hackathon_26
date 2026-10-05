# 01 Requirements and System Assumptions

## Executive Overview
The **SALESTORM 2026** platform is engineered to handle extreme flash sale events where massive surge traffic concentrates on a micro-inventory of high-demand items. This document defines the explicit Functional Requirements (FR), Non-Functional Requirements (NFR), system assumptions, operational constraints, and the formal matrix of Hard Architectural Guarantees versus Service Level Agreement (SLA) targets.

---

## 1. Functional Requirements (FR)

### FR-01: Product Discovery & Flash Sale Catalog
- **FR-01.1**: The system shall serve flash sale product catalog details (title, description, regular price, sale price, banner assets, stock status) to incoming users.
- **FR-01.2**: Catalog view must display near-real-time inventory availability (e.g., "In Stock", "Low Stock", "Sold Out") without issuing synchronous reads to the primary transactional database under peak load.

### FR-02: Flash Sale Inventory Check & Temporary Reservation
- **FR-02.1**: When a user clicks "Buy Now", the system shall validate user authenticity and check inventory availability in a concurrency-safe manner.
- **FR-02.2**: If inventory is available ($\le 100$ units remaining), the system shall atomically decrement available inventory by 1 and create a temporary **Inventory Reservation** linked to the user account.
- **FR-02.3**: Every reservation shall issue a cryptographically signed `reservation_token` with an immutable **10-minute time-to-live (TTL)**.
- **FR-02.4**: If the flash sale inventory reaches 0, all subsequent reservation requests must immediately receive an HTTP 409 Conflict / "Sold Out" response within sub-50ms without hitting the primary database.

### FR-03: Reservation Expiry & Automatic Inventory Release
- **FR-03.1**: If a user fails to complete checkout and payment authorization within the 10-minute window, the `reservation_token` shall expire.
- **FR-03.2**: An automated background worker process (triggered via Redis Key Expiry notifications and a distributed Delay Queue) shall release the reserved unit back into the available pool, allowing waiting customers to claim it.
- **FR-03.3**: The system must enforce strict state transitions: expired reservations cannot be converted into completed orders.

### FR-04: Idempotent Checkout & Payment Authorization
- **FR-04.1**: The Checkout Service shall validate the active `reservation_token` before accepting payment details.
- **FR-04.2**: The Payment Service must accept a client-generated or server-assigned `idempotency_key` (UUID v4 + UserID + ReservationID).
- **FR-04.3**: Repeated requests with the exact same `idempotency_key` (due to network retries, double-clicks, or Gateway timeouts) must return the original payment status without double-charging the user's payment instrument.
- **FR-04.4**: Payment integration with third-party gateways (Stripe, Razorpay, Adyen) must support asynchronous webhooks and polling reconciliation for pending/soft-failing payments.

### FR-05: Order Creation & Lifecycle Management
- **FR-05.1**: Upon verified payment authorization, the system shall transition the temporary reservation into a confirmed `ORDER_CREATED` state.
- **FR-05.2**: The Order Service shall manage state transitions: `PENDING` $\rightarrow$ `RESERVED` $\rightarrow$ `PAID` $\rightarrow$ `PROCESSING` $\rightarrow$ `FULFILLED` $\rightarrow$ `SHIPPED` $\rightarrow$ `DELIVERED` (or `CANCELLED` / `REFUNDED`).
- **FR-05.3**: Order state transitions must be event-driven, published to an append-only event bus (Kafka / RabbitMQ).

### FR-06: Fulfillment, Shipment & Delivery Tracking
- **FR-06.1**: Confirmed orders shall be enqueued for warehouse management system (WMS) pick-pack-ship operations.
- **FR-06.2**: The Fulfillment Service shall update tracking numbers and carrier details asynchronously.
- **FR-06.3**: Customers shall be able to query the exact real-time tracking status of their orders.

### FR-07: Real-Time Customer Notifications
- **FR-07.1**: The system shall emit multi-channel notifications (SMS, Email, Push) upon key events: Reservation Secured, Payment Success, Order Shipped, and Reservation Expired.

---

## 2. Non-Functional Requirements (NFR)

### NFR-01: Extreme Concurrency & Scalability
- **Target Load**: Handle **10,000 concurrent incoming requests** hitting the "Buy Now" endpoint within a 1-second burst window (10,000 RPS peak).
- **Extensibility Target**: Design must scale horizontally up to **500,000 concurrent requests** via Virtual Waiting Room edge traffic shedding.

### NFR-02: Latency & Performance SLAs
- **Reservation API Latency**: $p99 < 200\text{ ms}$, $p95 < 80\text{ ms}$, $p50 < 20\text{ ms}$.
- **Catalog View API Latency**: $p99 < 50\text{ ms}$ (served from CDN / Redis Read Replicas).
- **Payment & Order Creation Latency**: $p99 < 1500\text{ ms}$ (including third-party payment gateway roundtrips).

### NFR-03: Financial & Data Integrity (Zero Oversell Guarantee)
- **Absolute Inventory Ceiling**: Under no circumstances shall the system sell more than the allocated stock (**Maximum 100 successful sales** out of 10,000 concurrent requests).
- **No Duplicate Payments**: Strict **at-most-once** payment charge execution under all failure modes (gateway timeout, server crash, client disconnect).

### NFR-04: High Availability & Resilience
- **Service Availability Target**: **99.99% uptime** during flash sale windows.
- **Fault Tolerance**: The failure of secondary services (Notification, Analytics, Recommendation) must not impair the core Purchase $\rightarrow$ Payment $\rightarrow$ Order path.
- **Self-Healing Recovery**: In the event of a 30-second Order Service crash, the system must buffer incoming events in persistent message queues and reconcile state automatically upon recovery without dropping orders or double-charging.

### NFR-05: Security & Compliance
- **Authentication**: JWT / OAuth2 access tokens required for all mutating endpoints.
- **Rate Limiting**: IP-level and User-level rate limiting at API Gateway (1 request per user per 3 seconds for reservation endpoint).
- **PCI-DSS Compliance**: No raw credit card data or CVV stored in system databases. Tokenized payment signatures only.

---

## 3. Assumptions & Operational Constraints

1. **Flash Sale SKU Parameters**:
   - Total Units Available: **100 units**
   - Flash Sale Start Time: Fixed schedule (T-00:00:00)
   - Max Items Per User: **1 unit** per unique User ID.
2. **User Behavior**:
   - Total Active Competitors: **10,000 unique users** pressing "Buy Now" simultaneously at T-00:00:00.
   - User Drop-off Rate: ~30% of users who successfully reserve inventory may abandon checkout or experience payment card rejections.
3. **Infrastructure & Network Environment**:
   - AWS / GCP Cloud Deployment on Managed Kubernetes (EKS / GKE).
   - Third-Party Payment Gateway SLA: $p99 = 800\text{ ms}$, intermittent failure rate up to 2%.
   - Network Partitions: Intermittent 500ms–2000ms latency spikes between microservices can occur.

---

## 4. Hard Architectural Guarantees vs. SLA Targets

| Dimension | Hard Guarantee (Zero Tolerance) | Target SLA / Soft Target | Failure Handling Mechanism |
| :--- | :--- | :--- | :--- |
| **Inventory Cap** | **$\le 100$ total confirmed sales**. Zero oversell allowed under any concurrency level. | 100% stock utilization (all 100 sold if 100 buyers exist). | Atomic Redis Lua script + RDBMS Version OCC (`WHERE version = v AND available > 0`). |
| **Payment Integrity** | **Zero duplicate charges** per reservation. | Payment processing time $< 1.5\text{s}$. | Transactional Inbox/Outbox + Payment Gateway Idempotency Token. |
| **Reservation TTL** | **Strict 10-minute expiration** window. | Cleanup executed within $< 1\text{s}$ of expiry. | Redis Key Expiry Notifications + Delay Queue Worker Reconciliation. |
| **System Uptime** | **Core Flash Gateway available** during burst. | 99.99% overall platform uptime. | API Gateway Token Bucket Rate Limiter + Circuit Breakers (Resilience4j). |
| **Data Consistency** | **Eventual Consistency** across fulfillment within $< 5\text{s}$. | Real-time status update to user dashboard within $< 500\text{ms}$. | Orchestrated Saga Pattern with persistent Kafka topics & retry queues. |

---
*Document Version: 1.0.0 — SALESTORM 2026 Core Specifications*
