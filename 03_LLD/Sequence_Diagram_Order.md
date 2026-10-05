# 03.4 Low-Level Design — Sequence Diagram: Order Lifecycle & Crash Recovery

## Overview
This sequence diagram illustrates the **Order Lifecycle & Saga Orchestration** workflow. It details how confirmed payment events are consumed, how orders are persisted using Optimistic Concurrency Control (OCC), how fulfillment is triggered, and how the system recovers seamlessly during a **30-second Order Service crash**.

---

## 1. Happy Path Order Creation & Saga Orchestration

```mermaid
sequenceDiagram
    autonumber
    participant Kafka as 📡 Kafka Event Bus
    participant Saga as 📦 Order Saga Coordinator
    participant OrderDB as 🐘 Order DB (PostgreSQL)
    participant InvSvc as ⚡ Flash Inventory Service
    participant WMS as 🚚 Fulfillment Service (WMS)
    participant Notif as ✉️ Notification Worker

    %% Step 1: Payment Authorized Event Trigger
    Kafka->>Saga: Consume Event: payment.authorized {payment_id, reservation_id, user_id, amount}
    
    Note over Saga: Initiate Order Creation Saga
    Saga->>OrderDB: BEGIN TX
    Saga->>OrderDB: SELECT * FROM inventory_reservations WHERE reservation_id = ... FOR UPDATE
    Saga->>OrderDB: UPDATE inventory_reservations SET status = 'CONFIRMED' WHERE version = v
    Saga->>OrderDB: INSERT INTO orders (order_id, user_id, status='PAID', version=1)
    Saga->>OrderDB: INSERT INTO outbox_events (event_type='order.created', payload=...)
    Saga->>OrderDB: COMMIT TX

    %% Step 2: Publish Order Created Event
    Saga->>Kafka: Publish Event: order.created {order_id, user_id, items, shipping_addr}

    %% Step 3: Parallel Asynchronous Consumers
    par Fulfillment Processing
        Kafka->>WMS: Consume Event: order.created
        WMS->>WMS: Reserve Warehouse Physical Item & Print Shipping Label
        WMS->>Kafka: Publish Event: order.fulfilled {order_id, tracking_number: "TRK_88192"}
    and Customer Notification
        Kafka->>Notif: Consume Event: order.created
        Notif->>Notif: Dispatch Purchase Confirmation Email & SMS
    end

    %% Step 4: Final State Update
    Kafka->>Saga: Consume Event: order.fulfilled
    Saga->>OrderDB: UPDATE orders SET status = 'PROCESSING', tracking_number = 'TRK_88192'
```

---

## 2. Order Service 30-Second Outage & Self-Healing Recovery Sequence

This scenario demonstrates system resilience when the **Order Service container crashes for 30 seconds** right after the Payment Service publishes `payment.authorized`.

```mermaid
sequenceDiagram
    autonumber
    participant PaySvc as 💳 Payment Service
    participant Kafka as 📡 Kafka Event Bus (Topic: payment.authorized)
    participant OrderSvc as 📦 Order Service (Pod 1 & Pod 2)
    participant OrderDB as 🐘 Order DB
    participant ReconCron as 🔄 Saga Reconciliation Cron Worker

    PaySvc->>Kafka: Publish Event: payment.authorized {payment_id: "PAY_991", reservation_id: "RES_101"}
    
    Note over OrderSvc: 💥 OUTAGE OCCURS (t=0s to t=30s)<br/>Order Service Containers Crash (OOM / Node Fail)
    Kafka--xOrderSvc: Message sits safely in Kafka Topic Log (Uncommited Offset)

    Note over Kafka: Kafka Retention Policy holds events safely for 7 Days.<br/>No data loss! Zero dropped orders.

    Note over OrderSvc: 🔄 RECOVERY (t=35s)<br/>Kubernetes Kubelet restarts Order Service Pods
    OrderSvc->>Kafka: Re-connect to Consumer Group 'order-saga-group'
    Kafka-->>OrderSvc: Deliver Uncommited Message (Offset #48921: payment.authorized)
    
    OrderSvc->>OrderDB: SELECT * FROM orders WHERE reservation_id = 'RES_101'
    
    alt Order Record Found (Already processed before crash)
        OrderSvc->>Kafka: Commit Offset #48921 (Idempotent replay protection)
    else Order Record Missing (Needs creation)
        OrderSvc->>OrderDB: INSERT INTO orders (order_id, reservation_id, status='PAID')
        OrderSvc->>Kafka: Publish Event: order.created
        OrderSvc->>Kafka: Commit Offset #48921
    end

    %% Fallback Safety Net
    Note over ReconCron: 🛡️ Fallback Reconciliation Cron (Runs every 2 mins)
    ReconCron->>OrderDB: SELECT * FROM payments WHERE status='AUTHORIZED' AND payment_id NOT IN (SELECT payment_id FROM orders)
    opt Found Orphaned Payments
        ReconCron->>OrderSvc: Trigger Manual Saga Recovery for Orphaned Payment
    end
```

---

## 3. Resilience Guarantees & Disaster Recovery Metrics

1. **Zero Message Loss**: Kafka persistent append-only logs store uncommitted offsets. Even during container crashes, events remain durable on disk across replicas (`min.insync.replicas = 2`).
2. **At-Least-Once Delivery + At-Most-Once Execution**: Order consumers verify database state (`reservation_id` uniqueness) before applying state transitions, ensuring idempotency upon crash recovery.
3. **Recovery Time Objective (RTO)**: $< 45\text{ seconds}$ (Kubernetes pod restart + Kafka consumer group rebalance time).
4. **Recovery Point Objective (RPO)**: $0\text{ seconds}$ (No state lost due to WAL persistence and transactional outbox).

---
*Document Version: 1.0.0 — SALESTORM 2026 Order Sequence Design*
