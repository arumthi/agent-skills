# DDD Service Boundaries & Bounded Contexts Template

**Project Name**: [Your Project Name]  
**Architecture Date**: [Date]  
**Architect**: [Your Name]  

---

## Overview

Document the Domain-Driven Design (DDD) bounded contexts identified from your legacy SOAP system. Each bounded context maps to one microservice.

---

## 1. Domain Map

Visualize all bounded contexts and their relationships.

```
┌──────────────────────────────────────────────────────────────────────┐
│                      MICROSERVICES ECOSYSTEM                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│         ┌─────────────────────────────────────────┐                │
│         │         API GATEWAY                     │                │
│         │   (External client entry point)         │                │
│         └────────┬─────────────────────────┬──────┘                │
│                  │                         │                       │
│        ┌─────────▼──────────┐   ┌──────────▼──────────┐           │
│        │  Order Service     │   │ Inventory Service   │           │
│        │  (Order Context)   │──▶│  (Inventory Ctx)    │           │
│        └─────────┬──────────┘   └──────────┬──────────┘           │
│                  │                         │                       │
│        ┌─────────▼──────────┐   ┌──────────▼──────────┐           │
│        │ Payment Service    │   │Notification Service │           │
│        │(Payment Context)   │   │(Notification Ctx)   │           │
│        └────────────────────┘   └─────────────────────┘           │
│                                                                     │
│         ┌──────────────────────────────────┐                      │
│         │      Event Bus (Kafka/RabbitMQ)  │                      │
│         │   (Async communication layer)    │                      │
│         └──────────────────────────────────┘                      │
│                                                                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Bounded Context #1: Order Management

**Microservice Name**: order-service  
**Team Owner**: [Team name]  
**Domain Responsibility**: Manage customer orders from creation through fulfillment

### 2.1 Aggregate Roots

**Aggregate Root**: Order

```
Order Aggregate
├── Aggregate Root: Order {
│   ├── Identity: orderId (String)
│   ├── Attributes:
│   │   ├── customerId (String) - reference only, don't own
│   │   ├── status (OrderStatus enum)
│   │   ├── orderDate (Timestamp)
│   │   ├── totalAmount (Money)
│   │   └── shippingAddress (Address value object)
│   ├── Child Entities:
│   │   └── OrderItem {
│   │       ├── Identity: orderItemId
│   │       ├── productId (reference to Inventory service)
│   │       ├── quantity
│   │       ├── unitPrice
│   │       └── lineTotal
│   └── Business Rules:
│       ├── Order must have at least one item
│       ├── Status valid transitions: PENDING → CONFIRMED → SHIPPED → DELIVERED
│       ├── Can only cancel orders in PENDING or CONFIRMED status
│       └── Total amount = sum of all line items + shipping
└── Repository Method: OrderRepository {
    ├── Order findById(String orderId)
    ├── Void save(Order order)
    ├── List<Order> findByCustomerId(String customerId)
    └── Void delete(String orderId)
}
```

### 2.2 Ubiquitous Language

Terms that must be consistent across code, documentation, and team discussions:

| Term | Definition | Example |
|------|-----------|---------|
| Order | Customer's request for one or more items | "ORDER-12345" |
| OrderStatus | Current state of an order | PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED |
| OrderItem | Individual product within an order | Product ID "PROD-001", Quantity 2 |
| ShippingAddress | Where order should be delivered | Street, City, State, ZIP |
| BackOrder | Order for items currently out of stock | Scheduled shipment when restocked |

### 2.3 Domain Events

Events published by this bounded context:

```
Event: OrderCreated
├── When: Customer initiates new order
├── Data: { orderId, customerId, items, totalAmount, shippingAddress }
├── Subscribers: PaymentService, InventoryService, NotificationService
└── Guarantee: At-least-once delivery (via Kafka)

Event: OrderConfirmed
├── When: Payment received and inventory reserved
├── Data: { orderId, paymentId, reservationId }
├── Subscribers: InventoryService, NotificationService
└── Guarantee: At-least-once delivery

Event: OrderShipped
├── When: Order dispatched to customer
├── Data: { orderId, trackingNumber, shipmentDate }
├── Subscribers: NotificationService
└── Guarantee: At-least-once delivery

Event: OrderCancelled
├── When: Customer or system cancels order
├── Data: { orderId, reason, cancelledDate }
├── Subscribers: PaymentService, InventoryService, NotificationService
└── Guarantee: At-least-once delivery
```

### 2.4 Domain Services

Services (beyond single aggregate responsibility):

```
Service: OrderService.createOrder()
├── Input: CreateOrderRequest { customerId, items, shippingAddress }
├── Algorithm:
│   1. Create Order aggregate (status: PENDING)
│   2. Validate order (items not empty, prices correct)
│   3. Save to repository
│   4. Publish OrderCreated event
│   5. Return orderId
├── Transaction Boundary: Single transaction covers steps 1-3
├── Error Handling:
│   ├── ValidationException if items empty
│   ├── InsufficientInventoryException if stock unavailable
│   └── PaymentFailedException if payment fails
└── Idempotency: Use idempotencyKey to prevent duplicates

Service: OrderService.updateOrderStatus()
├── Input: orderId, newStatus
├── Algorithm:
│   1. Load Order aggregate
│   2. Validate status transition (PENDING → CONFIRMED is allowed)
│   3. Update status
│   4. Save to repository
│   5. Publish appropriate event (OrderConfirmed, OrderShipped, etc.)
└── Error Handling:
    ├── OrderNotFoundException
    └── InvalidStateTransitionException

Service: OrderService.cancelOrder()
├── Input: orderId, reason
├── Algorithm:
│   1. Load Order aggregate
│   2. Check if cancellable (PENDING or CONFIRMED only)
│   3. Set status to CANCELLED
│   4. Save to repository
│   5. Publish OrderCancelled event (triggers inventory release, payment refund)
└── Error Handling:
    └── CannotCancelException if order already shipped
```

### 2.5 Database Owner

This service exclusively owns and manages these tables:

```sql
-- In order_service_db schema
├── ORDERS table
│   ├── ORDER_ID (PK)
│   ├── CUSTOMER_ID (reference only - no FK)
│   ├── STATUS (unique business rule constraint)
│   ├── TOTAL_AMOUNT
│   └── ...created_date, updated_date
│
└── ORDER_ITEMS table
    ├── ORDER_ITEM_ID (PK)
    ├── ORDER_ID (FK to ORDERS)
    ├── PRODUCT_ID (reference only - no FK)
    ├── QUANTITY
    └── UNIT_PRICE
```

**Key Constraint**: Only order-service modifies these tables. All other services access via REST API calls.

### 2.6 External Dependencies

**Synchronous (REST API calls)**:
```
order-service → inventory-service.checkAvailability(productId, qty)
  [Blocking call during order creation]
  [Timeout: 5 sec, Fallback: Reject order]
```

**Asynchronous (Event subscriptions)**:
```
← payment-service: PaymentSucceeded event
  [Transitions order from PENDING to CONFIRMED]

← inventory-service: StockReservationFailed event
  [Cancels order and rolls back payment]
```

### 2.7 Anti-Corruption Layer

If integrating with legacy SOAP OrderService (during migration):

```java
// Legacy adapter isolates new code from WSDL contract
class LegacySoapOrderAdapter {
  
  // Translate domain model to SOAP
  OrderSoapRequest adaptToSoap(Order order) {
    return new OrderSoapRequest()
      .customerId(order.customerId())
      .items(order.items().stream()
        .map(item -> new SoapOrderItem(
          item.productId(), item.quantity(), item.price()
        ))
        .collect(...))
      .build();
  }
  
  // Translate SOAP response back to domain
  Order adaptFromSoap(OrderSoapResponse response) {
    return Order.create(
      orderId: response.getOrderId(),
      customerId: response.getCustomerId(),
      status: OrderStatus.from(response.getStatus()),
      ...
    );
  }
}
```

---

## 3. Bounded Context #2: Inventory Management

**Microservice Name**: inventory-service  
**Team Owner**: [Team name]  
**Domain Responsibility**: Track product availability, manage stock levels and reservations

### 3.1 Aggregate Root

**Aggregate Root**: Product

```
Product Aggregate
├── Aggregate Root: Product {
│   ├── Identity: productId (String)
│   ├── Attributes:
│   │   ├── name (String)
│   │   ├── description (String)
│   │   └── unitPrice (Money)
│   ├── Child Entities:
│   │   └── StockMovement {
│   │       ├── Identity: movementId
│   │       ├── type (Enum: RECEIVED, SOLD, RESERVED, RELEASED)
│   │       ├── quantity (Number)
│   │       └── referenceId (Order ID or PO ID)
│   └── Business Rules:
│       ├── Current stock = sum(RECEIVED + RETURNED) - sum(SOLD + RESERVED + ADJUSTED)
│       ├── Cannot reserve more than available stock
│       ├── Reserved stock is held for 1 hour; auto-released if not confirmed
│       └── All stock movements are immutable (audit trail)
└── Value Object: InventoryLevel {
    ├── currentStock (Number)
    ├── reservedStock (Number)
    └── availableStock (computed: current - reserved)
}
```

### 3.2 Domain Events

```
Event: StockReserved
├── When: Order requests inventory reservation
├── Data: { productId, quantity, orderId, expiresAt }
├── Subscribers: OrderService
└── Guarantee: At-least-once delivery

Event: StockReservationFailed
├── When: Insufficient inventory to fulfill reservation
├── Data: { productId, requestedQuantity, availableQuantity, orderId }
├── Subscribers: OrderService
└── Guarantee: At-least-once delivery

Event: StockReleased
├── When: Order cancelled or reservation expires
├── Data: { productId, quantity, orderId }
├── Subscribers: Payment Service (for refund trigger)
└── Guarantee: At-least-once delivery
```

### 3.3 Database Owner

```sql
-- In inventory_service_db schema
├── PRODUCTS table
│   ├── PRODUCT_ID (PK)
│   ├── PRODUCT_NAME
│   ├── UNIT_PRICE
│   └── ...created_date, updated_date
│
└── STOCK_MOVEMENTS table (append-only event log)
    ├── MOVEMENT_ID (PK)
    ├── PRODUCT_ID (FK)
    ├── MOVEMENT_TYPE
    ├── QUANTITY
    ├── REFERENCE_ID (Order ID)
    └── CREATED_DATE
```

### 3.4 External Dependencies

**Asynchronous subscriptions**:
```
← order-service: OrderCreated event
  [Attempts to reserve stock for all order items]

← order-service: OrderCancelled event
  [Releases all reserved stock]
```

---

## 4. Bounded Context #3: Payment Processing

**Microservice Name**: payment-service  
**Domain Responsibility**: Process, track, and manage payment transactions

### 4.1 Aggregate Root: Payment

[Structure similar to above contexts]

---

## 5. Communication Patterns

### Pattern 1: Request-Driven Saga for Order Creation

```
Timeline:
T0 → OrderService receives CreateOrderRequest
T1 → OrderService creates Order aggregate (PENDING status)
T2 → OrderService publishes OrderCreated event to Kafka

T3 → InventoryService receives OrderCreated event
T4 → InventoryService reserves stock
T5 → If success: InventoryService publishes StockReserved event
     If failure: InventoryService publishes StockReservationFailed event

T6 → PaymentService receives OrderCreated event
T7 → PaymentService charges payment gateway
T8 → If success: PaymentService publishes PaymentSucceeded event
     If failure: PaymentService publishes PaymentFailed event

T9 → OrderService receives StockReserved + PaymentSucceeded events
T10 → OrderService updates Order status from PENDING to CONFIRMED
T11 → OrderService publishes OrderConfirmed event

Result: Order status is eventually CONFIRMED (once all async operations succeed)
```

### Pattern 2: Consistent Failure Handling

```
Event: OrderCancelled is published (triggered by inventory or payment failure)

All services subscribe to OrderCancelled:
├── InventoryService: Releases any reserved stock
├── PaymentService: Refunds any charged payment
└── NotificationService: Sends cancellation email to customer
```

---

## 6. Integration Rules

**Contracts Between Services**:

| Service A | Service B | Contract | Type |
|-----------|-----------|----------|------|
| OrderService | InventoryService | checkAvailability(productId, qty) → Boolean | REST Call |
| OrderService | InventoryService | OrderCreated event | Event Pub/Sub |
| OrderService | PaymentService | OrderCreated event | Event Pub/Sub |
| All Services | NotificationService | Event subscriptions | Event Pub/Sub |

---

## 7. Deployment Topology

Each microservice has its own:

- **Git Repository**: [repo-url]
- **Docker Image**: [registry]/[service-name]:latest
- **Kubernetes Namespace**: [namespace]
- **Database**: Separate Oracle schema or database
- **Helm Chart**: helm-charts/[service-name]/
- **CI/CD Pipeline**: GitHub Actions / GitLab CI

---

## 8. Data Consistency Guarantees

| Scenario | Consistency Model | Guarantee |
|----------|---|---|
| Order creation + payment | Eventual | Both succeed or both rolled back |
| Order creation + inventory | Eventual | Both succeed or both compensated |
| Order status updates | Immediate | Single transaction in order-service |
| Cross-service query | Eventually consistent | May see stale data briefly |

---

## Document Control

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.0 | [Date] | [Name] | Initial design |
| 1.1 | [Date] | [Name] | [Change] |
