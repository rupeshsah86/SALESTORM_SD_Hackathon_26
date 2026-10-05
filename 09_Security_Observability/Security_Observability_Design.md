# 09 Security & Observability Design

## Overview
This document defines the zero-trust security framework, payment compliance standards, distributed tracing topology, real-time metrics collection, and alerting rules for **SALESTORM 2026**.

---

## 1. Security Architecture & Compliance

### 1. Authentication & Authorization Framework
- **JWT / OAuth2 Access Tokens**: Requests to mutating endpoints (`/reservations`, `/checkout`, `/payments`) require a cryptographically signed Bearer JWT token issued by the Identity Provider.
- **Stateless Claims Verification**: Kong API Gateway verifies JWT signatures using Public Keys (JWKS endpoint) at the edge, rejecting invalid or expired tokens in $< 2\text{ms}$.

### 2. Payment Data Security (PCI-DSS Compliance)
- **Zero Cardholder Storage**: Raw Primary Account Numbers (PAN) and Card Verification Values (CVV) **never touch SALESTORM servers or databases**.
- **Tokenization**: Client applications interact directly with Payment Gateway SDKs (Stripe Elements / Razorpay Checkout) to exchange credit card data for a single-use `payment_token`.

### 3. Webhook Signature Verification
- Incoming payment gateway webhooks are validated using HMAC-SHA256 signatures (`X-Stripe-Signature` header) matched against a shared secret to prevent spoofing attacks.

```python
# HMAC Webhook Verification
import hmac, hashlib

def verify_webhook_signature(payload_bytes, sig_header, secret):
    expected_sig = hmac.new(secret.encode(), payload_bytes, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected_sig, sig_header)
```

---

## 2. Observability Architecture (Metrics, Logs, Traces)

```
  Microservices (Go / Java / Node)
         │
         ├──> OpenTelemetry Collector (Traces & Metrics) ──> Jaeger / Grafana Tempo
         │
         ├──> Prometheus Exporter (/metrics) ──────────────> Prometheus ──> Grafana
         │
         └──> FluentBit / Vector (Structured JSON Logs) ───> Elasticsearch / Loki
```

### 1. Key Performance Indicators & Metrics (Prometheus)

| Metric Name | Type | Description | Target Threshold |
| :--- | :--- | :--- | :--- |
| `salestorm_reservation_requests_total` | Counter | Total incoming purchase reservation requests. | Monitored for spike detection. |
| `salestorm_stock_remaining_count` | Gauge | Real-time stock count in Redis. | Drops from 100 to 0. |
| `salestorm_reservation_latency_seconds` | Histogram | $p99$ latency of Lua script reservation API. | $p99 < 200\text{ms}$ |
| `salestorm_payment_idempotency_hits_total` | Counter | Total duplicate payment requests intercepted. | Identifies user double-clicks. |
| `salestorm_circuit_breaker_state` | Gauge | Status of payment gateway circuit (0=Closed, 1=Open). | Alert if Open $> 30\text{s}$. |

### 2. Distributed Tracing Topology (W3C Trace Context)
- Every incoming HTTP request receives a unique `traceparent` header at Kong API Gateway:
  `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
- Passed across gRPC metadata, HTTP headers, and Kafka event headers to allow end-to-end trace visualization in Jaeger.

---

## 3. Real-Time Alerting Rules (Alertmanager)

```yaml
groups:
  - name: salestorm_critical_alerts
    rules:
      - alert: HighOversellRisk
        expr: salestorm_stock_sold_total > 100
        for: 0m
        labels:
          severity: CRITICAL
        annotations:
          summary: "CRITICAL: Stock count exceeded 100 units limit!"

      - alert: PaymentGatewayCircuitOpen
        expr: salestorm_circuit_breaker_state{component="payment"} == 1
        for: 1m
        labels:
          severity: WARNING
        annotations:
          summary: "Payment gateway circuit breaker is OPEN."
```

---
*Document Version: 1.0.0 — SALESTORM 2026 Security & Observability*
