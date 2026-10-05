# 05 API Specifications & Event Schemas

## Overview
This document defines the formal RESTful API Endpoints, HTTP Status Codes, Error Schemas, and Kafka Event Payloads governing the **SALESTORM 2026** platform.

---

## 1. REST API Endpoint Summary

| Method | Endpoint Path | Description | Auth Required | Expected Latency |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/v1/products/{id}/flash-sale` | Fetch Flash Product Info & Real-Time Stock Status | Optional | $< 20\text{ms}$ |
| `POST`| `/api/v1/reservations` | Reserve Flash Inventory (10-Minute TTL Token) | Bearer JWT | $< 50\text{ms}$ |
| `POST`| `/api/v1/orders/checkout` | Submit Order using Reservation Token | Bearer JWT | $< 150\text{ms}$ |
| `POST`| `/api/v1/payments` | Process Idempotent Payment Authorization | Bearer JWT | $< 800\text{ms}$ |
| `GET` | `/api/v1/orders/{id}/tracking` | Query Order Status & Fulfillment Progress | Bearer JWT | $< 30\text{ms}$ |
| `POST`| `/api/v1/payments/webhook` | Webhook Receiver for Gateway Callbacks | HMAC Sig | $< 50\text{ms}$ |

---

## 2. API Endpoint Specifications

### Endpoint 1: Reserve Inventory
`POST /api/v1/reservations`

#### Headers
```http
Authorization: Bearer <JWT_ACCESS_TOKEN>
Content-Type: application/json
X-User-ID: usr_882194a2
```

#### Request Payload
```json
{
  "product_id": "prod_flash_2026_01",
  "quantity": 1
}
```

#### Response: HTTP 201 Created (Success Path)
```json
{
  "status": "SUCCESS",
  "data": {
    "reservation_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "product_id": "prod_flash_2026_01",
    "user_id": "usr_882194a2",
    "reservation_token": "RES_1728123400_991284",
    "expires_at": "2026-10-05T09:44:00Z",
    "expires_in_seconds": 600
  }
}
```

#### Response: HTTP 409 Conflict (Sold Out Path)
```json
{
  "status": "ERROR",
  "error": {
    "code": "FLASH_SALE_SOLD_OUT",
    "message": "All 100 units for this flash sale have been claimed.",
    "timestamp": "2026-10-05T09:34:01Z"
  }
}
```

---

### Endpoint 2: Process Idempotent Payment
`POST /api/v1/payments`

#### Headers
```http
Authorization: Bearer <JWT_ACCESS_TOKEN>
Content-Type: application/json
X-Idempotency-Key: IDEM_USR8821_RES9912_V1
```

#### Request Payload
```json
{
  "reservation_token": "RES_1728123400_991284",
  "amount": 99.00,
  "currency": "USD",
  "payment_method": {
    "type": "CREDIT_CARD",
    "token": "tok_stripe_test_card_11829"
  }
}
```

#### Response: HTTP 200 OK (Payment Authorized)
```json
{
  "status": "SUCCESS",
  "data": {
    "payment_id": "pay_7731a89d-421b-4392-8012-12093849182a",
    "reservation_token": "RES_1728123400_991284",
    "gateway_txn_id": "ch_3M891X29104",
    "amount": 99.00,
    "currency": "USD",
    "status": "AUTHORIZED",
    "idempotency_key": "IDEM_USR8821_RES9912_V1"
  }
}
```

#### Response: HTTP 202 Accepted (Gateway Timeout - Asynchronous Reconciliation)
```json
{
  "status": "PENDING",
  "message": "Payment gateway processing timeout. Payment status is being verified asynchronously.",
  "data": {
    "payment_id": "pay_7731a89d-421b-4392-8012-12093849182a",
    "status": "PENDING_RECONCILIATION",
    "check_status_url": "/api/v1/payments/pay_7731a89d-421b-4392-8012-12093849182a"
  }
}
```

---

## 3. Kafka Async Event Schemas

### Event 1: `inventory.reserved`
- **Topic**: `salestorm.inventory.events`
- **Partition Key**: `product_id`
```json
{
  "event_id": "evt_11928491-0192-4102-9912-102938491029",
  "event_type": "INVENTORY_RESERVED",
  "aggregate_type": "INVENTORY",
  "aggregate_id": "prod_flash_2026_01",
  "timestamp": "2026-10-05T09:34:00Z",
  "payload": {
    "reservation_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "product_id": "prod_flash_2026_01",
    "user_id": "usr_882194a2",
    "reservation_token": "RES_1728123400_991284",
    "expires_at": "2026-10-05T09:44:00Z"
  }
}
```

### Event 2: `payment.authorized`
- **Topic**: `salestorm.payment.events`
- **Partition Key**: `reservation_id`
```json
{
  "event_id": "evt_99128472-3102-4182-8812-402938491011",
  "event_type": "PAYMENT_AUTHORIZED",
  "aggregate_type": "PAYMENT",
  "aggregate_id": "pay_7731a89d-421b-4392-8012-12093849182a",
  "timestamp": "2026-10-05T09:34:12Z",
  "payload": {
    "payment_id": "pay_7731a89d-421b-4392-8012-12093849182a",
    "reservation_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "user_id": "usr_882194a2",
    "amount": 99.00,
    "currency": "USD",
    "gateway_txn_id": "ch_3M891X29104"
  }
}
```

---
*Document Version: 1.0.0 — SALESTORM 2026 API Specifications*
