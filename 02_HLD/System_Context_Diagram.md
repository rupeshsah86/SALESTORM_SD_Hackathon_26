# 02.1 High-Level Design — System Context Diagram (C4 Level 1)

## Overview
The System Context Diagram defines the high-level boundary of the **SALESTORM 2026** platform, identifying human actors, external enterprise dependencies, and primary communication protocols (Synchronous REST/gRPC vs Asynchronous Event/Webhook queues).

---

## C4 System Context Diagram

```mermaid
graph TB
    %% Actors
    User["👤 Flash Sale Buyer<br/>[10,000 Concurrent Users]<br/>Web & Mobile Clients"]
    Admin["👨‍💼 E-Commerce Admin<br/>Inventory & Flash Sale Manager"]

    %% Central System Boundary
    subgraph SALESTORM ["⚡ SALESTORM 2026 Platform System Boundary"]
        SystemCore["🚀 SALESTORM Core Platform<br/>(Inventory, Checkout, Payment, Order, Fulfillment Engine)"]
    end

    %% External Systems
    PaymentGateway["💳 External Payment Gateway<br/>(Stripe / Razorpay / Adyen)<br/>[PCI-DSS Vault & Payment Authorization]"]
    LogisticsProvider["🚚 External Logistics Provider<br/>(FedEx / DHL / WMS API)<br/>[Package Pick, Pack & Shipping Tracking]"]
    NotificationGateway["✉️ External Notification Provider<br/>(Twilio / SendGrid)<br/>[SMS, Email & Push Notifications]"]
    IdentityProvider["🔐 External Identity Provider<br/>(Auth0 / Google OAuth2)<br/>[User Authentication & JWT Tokens]"]

    %% Interactions - User to Core
    User -- "1. Browse Flash Catalog (Sync REST / HTTPS)" --> SystemCore
    User -- "2. Click 'Buy Now' / Reserve Inventory (Sync REST / HTTPS)" --> SystemCore
    User -- "3. Submit Payment Token & Checkout (Sync REST / HTTPS)" --> SystemCore
    User -- "4. Track Order Status (Sync REST / HTTPS)" --> SystemCore

    %% Interactions - Admin to Core
    Admin -- "Provision Stock & Schedule Sale (Sync REST / HTTPS)" --> SystemCore

    %% System Core to External Systems
    SystemCore -- "Auth Token Verification (Sync HTTPS)" --> IdentityProvider
    SystemCore -- "Process Charge / Auth Payment (Sync REST / HTTPS)" --> PaymentGateway
    PaymentGateway -. "Payment Status Webhooks (Async HTTPS)" .-> SystemCore
    SystemCore -. "Dispatch Shipping Orders (Async REST / Webhooks)" .-> LogisticsProvider
    LogisticsProvider -. "Tracking Updates (Async Webhooks)" .-> SystemCore
    SystemCore -. "Dispatch Notifications (Async Event Trigger)" .-> NotificationGateway

    %% Styling
    classDef actorStyle fill:#2b5c8f,stroke:#1a365d,stroke-width:2px,color:#fff;
    classDef systemStyle fill:#1e293b,stroke:#0f172a,stroke-width:3px,color:#fff;
    classDef externalStyle fill:#475569,stroke:#334155,stroke-width:2px,color:#fff;

    class User,Admin actorStyle;
    class SystemCore systemStyle;
    class PaymentGateway,LogisticsProvider,NotificationGateway,IdentityProvider externalStyle;
```

---

## Interface Protocol & Interaction Breakdown

| Source | Target | Interaction Type | Data Payload / Protocol | Purpose & SLA Target |
| :--- | :--- | :--- | :--- | :--- |
| **Buyer** | **SALESTORM** | Synchronous | HTTPS / JSON / REST | Flash catalog view ($< 50\text{ms}$), Reservation ($< 200\text{ms}$), Checkout. |
| **SALESTORM** | **IdP (Auth0)** | Synchronous | HTTPS / OAuth2 JWT | Validate Bearer access tokens on incoming requests ($< 15\text{ms}$). |
| **SALESTORM** | **Payment Gateway** | Synchronous | HTTPS / REST | Credit Card Authorization & Capture with Idempotency Header ($< 800\text{ms}$). |
| **Payment Gateway**| **SALESTORM** | Asynchronous | HTTPS Webhook | Out-of-band payment confirmation or async failure settlement. |
| **SALESTORM** | **Logistics** | Asynchronous | Webhook / REST | Fulfillment creation after payment confirmation ($< 2\text{s}$ queue latency). |
| **SALESTORM** | **Notification** | Asynchronous | AMQP / REST API | Dispatch purchase confirmation SMS/Email off the critical purchase path. |

---

## High-Level System Boundaries & Key Bottlenecks

### 1. Ingress Surge Bottleneck
- **Challenge**: 10,000 customers hitting the exact same endpoint in $t = 0$.
- **Architectural Boundary Mitigation**: Cloudflare Edge WAF + Kong API Gateway enforces Token Bucket Rate Limiting and Virtual Waiting Room queuing before traffic reaches application containers.

### 2. Third-Party Payment Gateway Dependency
- **Challenge**: Third-party payment gateways can experience latency degradation or HTTP 504 timeouts.
- **Architectural Boundary Mitigation**: Payment Service uses Circuit Breaker (Resilience4j) with fallback to asynchronous webhooks and outbox reconciliation.

---
*Document Version: 1.0.0 — SALESTORM 2026 Context Design*
