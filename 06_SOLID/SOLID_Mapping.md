# 06 SOLID Principles Mapping

## Overview
This document demonstrates how the **SALESTORM 2026** platform applies all five **SOLID software design principles** across its microservice components, domain entities, interfaces, and architecture layers.

---

## 1. Single Responsibility Principle (SRP)

> *"A class or component should have one, and only one, reason to change."*

### Concrete Application in SALESTORM
- **`LuaReservationEngine`**: Responsible solely for executing atomic in-memory Redis inventory operations and generating cryptographically signed reservation tokens. It has no knowledge of database schema or payment gateways.
- **`PaymentGatewayAdapter`**: Responsible exclusively for translating generic domain payment requests into vendor-specific API structures (e.g., Stripe JSON payloads) and handling HTTP communication.
- **`OrderSagaCoordinator`**: Manages distributed order state transitions and compensation triggers. It delegates database writes to `OrderRepository` and event broadcasting to `OutboxPublisher`.
- **`OutboxRelayerWorker`**: Dedicated to polling the database `outbox_events` table and streaming events to Kafka. It has no domain business logic.

---

## 2. Open/Closed Principle (OCP)

> *"Software entities should be open for extension, but closed for modification."*

### Concrete Application in SALESTORM
- **Extensible Payment Gateway Architecture**: Core payment processing logic depends on the `IPaymentGatewayAdapter` abstraction.
- **Extension Example**: Adding a new payment provider (e.g., *Adyen* or *Apple Pay*) requires creating a new class `AdyenGatewayAdapter implements IPaymentGatewayAdapter` without changing a single line of code in `PaymentService` or `CheckoutFacade`.

```java
// Open for extension: Implement new adapter
public class AdyenGatewayAdapter implements IPaymentGatewayAdapter {
    @Override
    public PaymentResponse processCharge(PaymentRequest request) {
        // Adyen specific API integration
        return new PaymentResponse(PaymentStatus.AUTHORIZED, adyenTxnId);
    }
}
```

---

## 3. Liskov Substitution Principle (LSP)

> *"Objects of a superclass should be replaceable with objects of a subclass without affecting correctness."*

### Concrete Application in SALESTORM
- Any class implementing `IPaymentGatewayAdapter` (e.g., `StripeGatewayAdapter`, `RazorpayGatewayAdapter`, or `MockPaymentAdapter` for load testing) satisfies the strict behavioral contract.
- Every adapter throws unified domain exceptions (`PaymentDeclinedException`, `GatewayTimeoutException`) rather than leaky third-party vendor exceptions (e.g., `StripeException` or `RazorpayException`).

---

## 4. Interface Segregation Principle (ISP)

> *"Clients should not be forced to depend upon interfaces that they do not use."*

### Concrete Application in SALESTORM
- Instead of creating monolithic "God Interfaces" (e.g., `IInventoryService`), SALESTORM breaks interfaces into lean, focused contracts:

```java
// Fast read-only stock lookup for catalog display
public interface IStockReader {
    int getAvailableStock(String productId);
}

// High-speed write reservation contract for flash engine
public interface IStockReservationEngine {
    ReservationResult reserveStock(String productId, String userId, int ttlSeconds);
}

// Separate interface for inventory management admin operations
public interface IStockAdminManager {
    void allocateStock(String productId, int newQuantity);
    void auditStockDiscrepancies(String productId);
}
```

---

## 5. Dependency Inversion Principle (DIP)

> *"High-level modules should not depend on low-level modules. Both should depend on abstractions."*

### Concrete Application in SALESTORM
- `CheckoutService` (high-level orchestration) does not depend directly on concrete classes like `PostgresOrderRepository` or `KafkaEventPublisher`.
- It injects interface abstractions (`IOrderRepository`, `IEventBusPublisher`) via Constructor Dependency Injection.

```java
@Service
public class CheckoutService {
    private final IOrderRepository orderRepository; // Abstraction
    private final IEventBusPublisher eventPublisher; // Abstraction

    @Autowired
    public CheckoutService(IOrderRepository orderRepository, IEventBusPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }
}
```

---
*Document Version: 1.0.0 — SALESTORM 2026 SOLID Mapping*
