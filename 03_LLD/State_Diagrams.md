# 03.5 Low-Level Design — State Diagrams

## Overview
This document defines the formal finite state machines (FSM) governing the **Inventory Reservation**, **Order Lifecycle**, and **Payment Transaction** domains. State transitions enforce immutability, state validation rules, and explicit rollback triggers under failure.

---

## 1. Inventory Reservation State Machine

```mermaid
stateDiagram-v2
    [*] --> UNRESERVED : Product In Stock (Count > 0)

    UNRESERVED --> RESERVED : User executes "Buy Now"<br/>[Redis Lua script decrements stock & sets 10m TTL]
    
    state RESERVED {
        [*] --> PENDING_PAYMENT : Token Issued (RES_HASH)
        PENDING_PAYMENT --> PENDING_PAYMENT : Payment Gateway Processing
    }

    RESERVED --> CONFIRMED : Payment Authorized Event Received<br/>[DB OCC Update status='CONFIRMED']
    RESERVED --> EXPIRED : 10-Minute TTL Expires<br/>[Redis keyspace notification -> INCR stock]
    RESERVED --> RELEASED : Payment Soft Failed / User Cancelled<br/>[Manual release -> INCR stock]

    CONFIRMED --> [*] : Unit Permanently Sold
    EXPIRED --> UNRESERVED : Unit Restored to Available Pool
    RELEASED --> UNRESERVED : Unit Restored to Available Pool
```

### Transition Rules — Inventory Reservation
- **`UNRESERVED` $\rightarrow$ `RESERVED`**: Atomic execution in Redis Lua script. Emits `reservation_token` with 600s TTL.
- **`RESERVED` $\rightarrow$ `CONFIRMED`**: Terminal success transition upon `payment.authorized` event. Stock counter remains decremented; DB record set to `CONFIRMED`.
- **`RESERVED` $\rightarrow$ `EXPIRED`**: Triggered asynchronously via Redis key expiry listener when $t > 600\text{s}$. Atomically increments Redis stock (`INCR`).
- **`RESERVED` $\rightarrow$ `RELEASED`**: Triggered via explicit compensation when payment card is declined.

---

## 2. Order Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> DRAFT : User Initiates Checkout

    DRAFT --> PENDING_PAYMENT : Reservation Verified

    PENDING_PAYMENT --> PAID : Payment Authorized Event<br/>[Order Created in PostgreSQL via OCC]
    PENDING_PAYMENT --> CANCELLED : Reservation Expired / Payment Failed

    PAID --> PROCESSING : Order Sent to Warehouse WMS
    
    PROCESSING --> SHIPPED : Package Picked & Tracking Assigned
    
    SHIPPED --> DELIVERED : Delivery Confirmed by Carrier API

    PROCESSING --> CANCELLED : Warehouse Out-of-Stock / Defect
    PAID --> REFUNDED : Customer Cancelled / Admin Override

    CANCELLED --> [*] : Terminal State
    DELIVERED --> [*] : Terminal State
    REFUNDED --> [*] : Terminal State
```

### Transition Rules — Order Lifecycle
- **Invalid State Transition Guard**: An order in `SHIPPED` or `DELIVERED` state can never transition back to `PENDING_PAYMENT` or `CANCELLED`.
- **Compensation Guard**: Transition to `REFUNDED` automatically triggers an asynchronous payment reversal event to the Payment Gateway and dispatches a notification.

---

## 3. Payment Transaction State Machine

```mermaid
stateDiagram-v2
    [*] --> INITIATED : Client Submits Payment Request

    INITIATED --> PROCESSING : Idempotency Check Passed & Lock Acquired

    PROCESSING --> AUTHORIZED : Payment Gateway Succeeded (200 OK)
    PROCESSING --> PENDING_RECONCILIATION : Gateway Timeout (HTTP 504)
    PROCESSING --> FAILED : Payment Declined / Invalid Card (402)

    PENDING_RECONCILIATION --> AUTHORIZED : Webhook Received (Status = Succeeded)
    PENDING_RECONCILIATION --> FAILED : Reconciliation Cron Query (Status = Failed)

    AUTHORIZED --> CAPTURED : Settlement Complete
    AUTHORIZED --> REFUNDED : Saga Compensation / User Cancellation

    FAILED --> [*] : Terminal State
    CAPTURED --> [*] : Terminal State
    REFUNDED --> [*] : Terminal State
```

---
*Document Version: 1.0.0 — SALESTORM 2026 State Diagram Specifications*
