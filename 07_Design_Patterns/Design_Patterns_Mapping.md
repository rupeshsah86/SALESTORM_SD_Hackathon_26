# 07 Design Patterns Mapping

## Overview
This document details the software design patterns implemented across the **SALESTORM 2026** platform to solve high-concurrency, resilience, payment idempotency, and distributed consistency challenges.

---

## 1. Summary of Applied Design Patterns

| Design Pattern | System Component | Primary Architectural Purpose |
| :--- | :--- | :--- |
| **Strategy Pattern** | Payment Service | Dynamically select payment gateway (Stripe vs Razorpay vs Mock) at runtime. |
| **Factory Pattern** | `PaymentGatewayFactory` | Instantiate payment gateway strategies based on currency and user location. |
| **State Pattern** | `OrderStateMachine` | Enforce valid state transitions for orders (`CREATED` $\rightarrow$ `PAID` $\rightarrow$ `SHIPPED`). |
| **Observer / Event Pattern**| Kafka Event Stream | Asynchronous notification of `inventory.reserved` and `payment.authorized`. |
| **Adapter Pattern** | Gateway Adapters | Wrap third-party SDK responses into uniform domain entities. |
| **Repository Pattern** | `OrderRepository` | Isolate database SQL Optimistic Locking logic from business domain. |
| **Facade Pattern** | `CheckoutFacade` | Provide a unified, simplified interface to catalog, inventory, and payment subsystems. |
| **Circuit Breaker Pattern** | `Resilience4j` Gateway Wrapper | Prevent cascade failure when external payment providers experience HTTP 504 latency. |
| **Transactional Outbox** | `OutboxPublisher` | Eliminate dual-write discrepancies between RDBMS transactions and Kafka messages. |
| **Saga Orchestrator** | `OrderSagaCoordinator` | Coordinate multi-service distributed transactions with automated compensations. |

---

## 2. Detailed Pattern Specifications & Implementation Examples

### 1. Strategy & Factory Pattern (Payment Provider Selection)

```java
// Strategy Interface
public interface IPaymentGatewayStrategy {
    PaymentResponse executePayment(PaymentRequest request);
    boolean supportsCurrency(String currency);
}

// Concrete Strategy A
@Component("STRIPE")
public class StripePaymentStrategy implements IPaymentGatewayStrategy {
    public PaymentResponse executePayment(PaymentRequest request) {
        // Stripe SDK Integration
        return new PaymentResponse("STRIPE_TXN_991", PaymentStatus.AUTHORIZED);
    }
    public boolean supportsCurrency(String currency) { return "USD".equalsIgnoreCase(currency); }
}

// Factory Class
@Component
public class PaymentGatewayFactory {
    private final Map<String, IPaymentGatewayStrategy> strategies;

    public PaymentGatewayFactory(List<IPaymentGatewayStrategy> strategyList) {
        strategies = strategyList.stream()
            .collect(Collectors.toMap(s -> s.getClass().getAnnotation(Component.class).value(), s -> s));
    }

    public IPaymentGatewayStrategy getStrategy(String providerName) {
        IPaymentGatewayStrategy strategy = strategies.get(providerName.toUpperCase());
        if (strategy == null) {
            throw new IllegalArgumentException("Unsupported Payment Gateway: " + providerName);
        }
        return strategy;
    }
}
```

---

### 2. Circuit Breaker Pattern (Resilience4j Protection)

- **Problem**: Third-party payment APIs can freeze or timeout under high surge load, starving application thread pools.
- **Solution**: Wrap external gateway HTTP calls with a **Resilience4j Circuit Breaker**.

```java
@Service
public class ResilientPaymentService {
    
    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "paymentGateway")
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> 
            paymentGatewayFactory.getStrategy(request.getGateway()).executePayment(request)
        );
    }

    // Fallback when Circuit is OPEN or Gateway Times Out (504)
    public CompletableFuture<PaymentResponse> paymentFallback(PaymentRequest request, Throwable t) {
        // Soft-fail: Mark payment as PENDING_RECONCILIATION & queue background webhook check
        return CompletableFuture.completedFuture(
            new PaymentResponse(null, PaymentStatus.PENDING_RECONCILIATION)
        );
    }
}
```

---

### 3. Transactional Outbox Pattern (Dual-Write Prevention)

- **Problem**: Updating a database (`payments` table) and publishing a Kafka message in separate calls leads to inconsistencies if the container crashes between the two calls.
- **Solution**: Save both the state update and the outgoing event in the **same RDBMS transaction**. A background CDC worker reads the `outbox_events` table and publishes to Kafka reliably.

```java
@Transactional
public PaymentResult processAndStorePayment(PaymentRequest request) {
    // 1. Update Payment Ledger
    Payment payment = new Payment(request.getAmount(), PaymentStatus.AUTHORIZED);
    paymentRepository.save(payment);

    // 2. Insert Outbox Event in SAME Transaction
    OutboxEvent event = new OutboxEvent(
        "PAYMENT", 
        payment.getId().toString(), 
        "payment.authorized", 
        objectMapper.writeValueAsString(payment)
    );
    outboxRepository.save(event);

    return PaymentResult.success(payment.getId());
}
```

---
*Document Version: 1.0.0 — SALESTORM 2026 Design Patterns Specification*
