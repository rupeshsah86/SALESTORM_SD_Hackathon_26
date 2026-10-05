# 03.2 Low-Level Design — Sequence Diagram: Purchase & Reservation

## Overview
This sequence diagram details the extreme-concurrency **Inventory Reservation** path when 10,000 customers hit the "Buy Now" endpoint simultaneously. It details the interaction between the API Gateway, Flash Inventory Microservice, Redis Cluster (executing an atomic Lua script), and the automated 10-minute expiry cleanup workflow.

---

## 1. Purchase & Inventory Reservation Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as 👤 Buyer (10,000 Concurrent Users)
    participant Gateway as 🛡️ API Gateway (Kong)
    participant InvSvc as ⚡ Inventory Microservice
    participant Redis as ⚡ Redis Cluster (Master)
    participant Kafka as 📡 Kafka Event Bus
    participant DBRelay as 🔄 DB Outbox Worker
    participant Postgres as 🐘 PostgreSQL (Order DB)

    %% Step 1: Request Ingress & Rate Limiting
    Buyer->>Gateway: POST /api/v1/reservations {product_id: "prod_101"} (Bearer JWT)
    Note over Gateway: 1. Token Bucket Rate Check<br/>2. Verify JWT Signature<br/>3. Enforce 1 req/user/3s
    
    alt Rate Limit Exceeded or Token Invalid
        Gateway-->>Buyer: HTTP 429 Too Many Requests / 401 Unauthorized
    else Request Approved
        Gateway->>InvSvc: gRPC ReserveInventory(user_id, product_id, ttl=600s)
    end

    %% Step 2: Atomic Execution in Redis
    Note over InvSvc,Redis: Execute Single-Threaded Redis Lua Script
    InvSvc->>Redis: EVALSHA lua_reserve_script 2 stock:prod_101 res:prod_101:user_id user_id 600

    alt Stock == 0 (Sold Out)
        Redis-->>InvSvc: Return CODE: -1 ("SOLD_OUT")
        InvSvc-->>Gateway: HTTP 409 Conflict {"error": "FLASH_SALE_SOLD_OUT"}
        Gateway-->>Buyer: HTTP 409 Conflict ("Flash sale items are fully reserved")
    else User Already Has Active Reservation
        Redis-->>InvSvc: Return CODE: -2 ("ALREADY_RESERVED")
        InvSvc-->>Gateway: HTTP 409 Conflict {"error": "DUPLICATE_RESERVATION"}
        Gateway-->>Buyer: HTTP 409 Conflict ("You already reserved a unit")
    else Stock > 0 (Success Path - Max 100 Users)
        Redis-->>InvSvc: Return CODE: 1, token: "RES_HASH_9912", remaining: 42
        
        %% Async Notification & Event Streaming
        InvSvc->>Kafka: Publish Event: inventory.reserved {reservation_id, user_id, expires_at}
        InvSvc->>DBRelay: Async enqueue reservation log to Outbox
        DBRelay->>Postgres: INSERT INTO inventory_reservations (status='PENDING', expires_at=NOW()+10m)
        
        InvSvc-->>Gateway: HTTP 201 Created {reservation_token: "RES_HASH_9912", expires_in: 600}
        Gateway-->>Buyer: HTTP 201 Created {reservation_token, expires_in: 600s, status: "RESERVED"}
    end
```

---

## 2. Production Redis Lua Script (Atomic Reservation Engine)

To guarantee **Zero Oversell** under 10,000 concurrent threads without database row locks, the system executes the following atomic Lua script inside Redis:

```lua
-- KEYS[1]: stock:product_id (String counter e.g., "100")
-- KEYS[2]: reservation:product_id:user_id (String token key with TTL)
-- ARGV[1]: user_id
-- ARGV[2]: ttl_seconds (e.g., 600)

local stock_key = KEYS[1]
local user_res_key = KEYS[2]
local user_id = ARGV[1]
local ttl = tonumber(ARGV[2])

-- Check if user already holds an active reservation
if redis.call("EXISTS", user_res_key) == 1 then
    return {-2, "ALREADY_RESERVED", 0}
end

-- Get current stock count
local current_stock = tonumber(redis.call("GET", stock_key) or "0")

if current_stock <= 0 then
    return {-1, "SOLD_OUT", 0}
end

-- Atomically decrement stock
local remaining_stock = redis.call("DECR", stock_key)

-- Generate secure reservation token hash
local token = "RES_" .. redis.call("TIME")[1] .. "_" .. math.random(100000, 999999)

-- Set reservation key with 10-minute TTL (600 seconds)
redis.call("SET", user_res_key, token, "EX", ttl)

-- Return Success Code 1, Reservation Token, and Remaining Stock
return {1, token, remaining_stock}
```

---

## 3. Reservation Expiry & Automatic Stock Release Sequence

If a customer reserves a unit but fails to complete payment within 10 minutes (600 seconds), the reservation automatically expires and releases the unit back into the available pool for other buyers.

```mermaid
sequenceDiagram
    autonumber
    participant Redis as ⚡ Redis Cluster
    participant ExpiryWorker as 🔄 Expiry Listener Worker
    participant Kafka as 📡 Kafka Event Bus
    participant Postgres as 🐘 PostgreSQL (Order DB)

    Note over Redis: 10-Minute TTL Expires on res:prod_101:user_8821
    Redis-->>ExpiryWorker: Keyspace Notification: __keyevent@0__:expired ("res:prod_101:user_8821")
    
    Note over ExpiryWorker: Extract product_id ("prod_101") & user_id ("user_8821")
    ExpiryWorker->>Redis: INCR stock:prod_101 (Atomically restore +1 stock unit)
    Redis-->>ExpiryWorker: Stock count incremented
    
    ExpiryWorker->>Kafka: Publish Event: inventory.released {product_id, user_id, reason: "EXPIRED"}
    ExpiryWorker->>Postgres: UPDATE inventory_reservations SET status = 'EXPIRED' WHERE reservation_id = ...
    
    Note over Kafka: Waiting Room Buyers notified of stock availability via WebSocket
```

---
*Document Version: 1.0.0 — SALESTORM 2026 Reservation Sequence Design*
