# 03.3 Low-Level Design — Sequence Diagram: Payment Processing & Idempotency

## Overview
This sequence diagram details the **Payment Service** execution pipeline. It models strict idempotency handling, third-party payment gateway integration (Stripe/Razorpay), HTTP 504 timeout recovery, duplicate submission protection, and the Transactional Outbox event publishing mechanism.

---

## 1. Idempotent Payment Authorization Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as 👤 Buyer
    participant Gateway as 🛡️ API Gateway
    participant PaySvc as 💳 Payment Microservice
    participant IdemStore as 🔑 Idempotency Ledger (Redis/Postgres)
    participant ExtPG as 🌐 External Payment Gateway (Stripe)
    participant PayDB as 🐘 Payment DB (Outbox)
    participant Kafka as 📡 Kafka Event Bus

    Buyer->>Gateway: POST /api/v1/payments {reservation_token, idempotency_key, payment_token}
    Gateway->>PaySvc: ProcessPayment(payload, Header: X-Idempotency-Key)

    %% Step 1: Check Idempotency Ledger
    PaySvc->>IdemStore: Query idempotency_key: "IDEM_USR88_RES9912"
    
    alt Idempotency Key Already Executed (Duplicate Request)
        IdemStore-->>PaySvc: Return Cached Response {status: "AUTHORIZED", payment_id: "PAY_7731"}
        PaySvc-->>Gateway: HTTP 200 OK (Cached Payment Payload)
        Gateway-->>Buyer: HTTP 200 OK {"status": "PAID", "message": "Duplicate request safely handled"}
    else New Request (First Attempt)
        PaySvc->>IdemStore: Acquire Lock idempotency_key (Status: PROCESSING, TTL: 60s)
        
        %% Step 2: Call External Gateway with Circuit Breaker
        Note over PaySvc,ExtPG: Execute Third-Party Charge via Circuit Breaker
        PaySvc->>ExtPG: POST /v1/charges {amount: 99.00, currency: "USD", idempotency_key}
        
        alt Payment Authorized (Success Path)
            ExtPG-->>PaySvc: HTTP 200 OK {charge_id: "ch_3M891", status: "succeeded"}
            
            %% Step 3: Transactional Outbox Atomic Write
            Note over PaySvc,PayDB: Atomic RDBMS Transaction
            PaySvc->>PayDB: BEGIN TX
            PaySvc->>PayDB: UPDATE idempotency_record SET status = 'COMPLETED', payload = ...
            PaySvc->>PayDB: INSERT INTO payments (payment_id, reservation_id, status='AUTHORIZED')
            PaySvc->>PayDB: INSERT INTO outbox_events (event_type='payment.authorized', payload=...)
            PaySvc->>PayDB: COMMIT TX
            
            %% Step 4: Async CDC Outbox Event Streaming
            PaySvc->>Kafka: Publish Event: payment.authorized {payment_id, reservation_id, user_id}
            PaySvc-->>Gateway: HTTP 200 OK {payment_id: "PAY_7731", status: "AUTHORIZED"}
            Gateway-->>Buyer: HTTP 200 OK {payment_id, status: "SUCCESS"}

        else Payment Gateway Timeout (HTTP 504 / Connection Drop)
            ExtPG--xPaySvc: HTTP 504 Gateway Timeout (Network Latency Spike)
            Note over PaySvc: Do NOT assume failure! Payment might be pending at Provider.
            PaySvc->>PayDB: INSERT INTO payments (status='PENDING_RECONCILIATION')
            PaySvc->>Kafka: Publish Event: payment.pending_reconciliation
            PaySvc-->>Gateway: HTTP 202 Accepted {payment_id, status: "PENDING", check_url: "/api/v1/payments/status"}
            Gateway-->>Buyer: HTTP 202 Accepted ("Payment processing in background. Verifying status.")

        else Payment Soft Fail / Card Declined
            ExtPG-->>PaySvc: HTTP 402 Payment Required {code: "card_declined"}
            PaySvc->>PayDB: UPDATE idempotency_record SET status = 'FAILED'
            PaySvc->>Kafka: Publish Event: payment.failed {reason: "CARD_DECLINED"}
            PaySvc-->>Gateway: HTTP 400 Bad Request {error: "PAYMENT_DECLINED"}
            Gateway-->>Buyer: HTTP 400 Bad Request ("Payment card declined. Please try another card.")
        end
    end
```

---

## 2. Webhook & Asynchronous Payment Reconciliation Sequence

When a payment gateway timed out (HTTP 504), the Payment Service relies on Webhook callbacks or a background polling cron worker to resolve the transaction state definitively.

```mermaid
sequenceDiagram
    autonumber
    participant ExtPG as 🌐 External Payment Gateway
    participant WebhookHandler as 💳 Payment Webhook Service
    participant PayDB as 🐘 Payment DB
    participant Kafka as 📡 Kafka Event Bus
    participant Saga as 📦 Order Saga Coordinator

    ExtPG->>WebhookHandler: POST /api/v1/payments/webhook {charge_id: "ch_3M891", status: "succeeded"}
    Note over WebhookHandler: Verify Webhook HMAC Signature (X-Stripe-Signature)

    WebhookHandler->>PayDB: SELECT status FROM payments WHERE gateway_txn_id = 'ch_3M891'
    
    alt Status Already AUTHORIZED / COMPLETED
        WebhookHandler-->>ExtPG: HTTP 200 OK (Idempotent Webhook Acknowledgment)
    else Status Is PENDING_RECONCILIATION
        WebhookHandler->>PayDB: BEGIN TX
        WebhookHandler->>PayDB: UPDATE payments SET status = 'AUTHORIZED' WHERE payment_id = ...
        WebhookHandler->>PayDB: INSERT INTO outbox_events (event_type='payment.authorized', ...)
        WebhookHandler->>PayDB: COMMIT TX
        
        WebhookHandler->>Kafka: Publish Event: payment.authorized
        Kafka->>Saga: Trigger Order Saga Continuation
        WebhookHandler-->>ExtPG: HTTP 200 OK
    end
```

---

## 3. Payment Failure Recovery & Circuit Breaker Logic

- **Circuit Breaker Thresholds**: Configured with Resilience4j ($50\%$ failure rate over a rolling 20-request window triggers `OPEN` state).
- **Fallback Action**: When the Payment Gateway circuit opens, the system rejects new payment authorizations gracefully with `HTTP 503 Service Unavailable`, preventing cascade failures and allowing the gateway to recover.

---
*Document Version: 1.0.0 — SALESTORM 2026 Payment Sequence Design*
