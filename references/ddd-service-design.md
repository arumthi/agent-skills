# Domain-Driven Design: Service Boundaries

## Overview

Domain-Driven Design (DDD) helps identify natural service boundaries by analyzing the business domain. This guide helps you decompose a monolithic SOAP application into cohesive microservices.

## Core DDD Concepts

### Bounded Context

A boundary within which a domain term has a specific meaning. Services typically map 1:1 to bounded contexts.

```
Example: "Order" has different meanings in different contexts:
- ORDER CONTEXT: An aggregation of customer intent, items, and fulfillment
- PAYMENT CONTEXT: An amount to be charged with associated transaction status
- INVENTORY CONTEXT: A claim on products to be reserved and picked

→ Create separate Order Service, Payment Service, Inventory Service
```

### Aggregate

A cluster of domain objects that can be treated as a single unit. Usually maps to a JPA entity with related entities.

```
Order Aggregate:
├── Order (root entity)
│   ├── orderId (identity)
│   ├── customerId
│   ├── status
│   ├── items: List<OrderItem>  ← child entity
│   ├── shippingAddress
│   └── shippingAddress
└── Should be treated as atomic unit during business transactions
```

### Ubiquitous Language

Domain-specific terms that should be consistent across code, documentation, and team discussions.

```
Order Service Ubiquitous Language:
- Order: Customer's request for products/services
- OrderItem: Individual product within an order
- OrderStatus: {PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED}
- Fulfillment: Process of picking, packing, shipping order
- Backorder: Order for out-of-stock items to be shipped later

Avoid generic terms like "transaction", "request", "job"
```

## Step 1: Identify Business Domains

From your legacy SOAP system, list all business activities:

```
From legacy OrderService, InventoryService, PaymentService, NotificationService:

Broad Domains:
1. Order Management
   - Create orders
   - Modify orders (update items, address)
   - Cancel orders
   - View order status

2. Payment Processing
   - Charge payment methods
   - Handle payment failures
   - Refund transactions

3. Inventory Management
   - Check product availability
   - Reserve stock for orders
   - Update stock counts
   - Handle backorders

4. Customer Management
   - Manage customer profiles
   - Track shipping addresses
   - Manage preferences

5. Notifications
   - Send order confirmations
   - Send shipment updates
   - Send payment receipts
```

## Step 2: Group Related Entities into Bounded Contexts

For each domain, identify aggregates and their relationships:

```
BOUNDED CONTEXT: Order Service
  Aggregates:
    - Order (root aggregate)
      Fields: orderId, customerId, orderDate, status, items, shippingAddress, totalAmount
      Child Entities: OrderItem (productId, quantity, price)
      Value Objects: Money (amount, currency), Address (street, city, state, zip)
      Events: OrderCreated, OrderConfirmed, OrderShipped, OrderCancelled
    
  Repositories:
    - OrderRepository (find order, save order, delete order)
    
  Services:
    - OrderService (create order, update order, cancel order)
    
  External Dependencies:
    - PaymentService (async call to charge payment)
    - InventoryService (async call to reserve stock)
    - NotificationService (async event for order confirmation)

BOUNDED CONTEXT: Inventory Service
  Aggregates:
    - Product (root)
      Fields: productId, name, description, currentStock, reservedStock
      Child Entities: StockMovement (for audit trail)
      Value Objects: InventoryLevel (quantity, unit)
      Events: StockReserved, StockReleased, StockAdjusted
    
  Repositories:
    - ProductRepository
    
  Services:
    - InventoryService (check availability, reserve stock, release stock)
    
  External Dependencies:
    - OrderService (receives order events)

BOUNDED CONTEXT: Payment Service
  Aggregates:
    - Payment (root)
      Fields: paymentId, orderId, amount, status, method, transactionId
      Value Objects: Money (amount, currency), PaymentMethod (card, bank, wallet)
      Events: PaymentInitiated, PaymentSucceeded, PaymentFailed, PaymentRefunded
    
  Repositories:
    - PaymentRepository
    
  Services:
    - PaymentService (process payment, handle refunds)
    
  External Dependencies:
    - Payment Gateway (Stripe, Square, etc.)
    - OrderService (receives order events)

BOUNDED CONTEXT: Notification Service
  Aggregates:
    - Notification (root)
      Fields: notificationId, customerId, type, status, sentDate
      Value Objects: NotificationContent (template, parameters)
      Events: NotificationSent, NotificationFailed
    
  Services:
    - NotificationService (send email, SMS, push)
    
  External Dependencies:
    - Email Provider (SendGrid)
    - SMS Provider (Twilio)
    - Event Bus (receives order, payment, inventory events)
```

## Step 3: Map Bounded Contexts to Microservices

Create a matrix of bounded contexts and their responsibilities:

```
Microservice Name: order-service
  Bounded Context: Order Management
  Responsibilities:
    - Accept new orders
    - Validate order completeness
    - Manage order lifecycle (PENDING → CONFIRMED → SHIPPED → DELIVERED)
    - Handle cancellations
    - Query order history
  
  Database: order_service_db (Oracle)
    Tables: orders, order_items, order_history
    Owned by: order-service (no other service modifies these tables)
  
  APIs (REST):
    POST /api/v1/orders (create order)
    GET /api/v1/orders/{id} (fetch order)
    PATCH /api/v1/orders/{id} (update order)
    DELETE /api/v1/orders/{id} (cancel order)
    GET /api/v1/orders?filter=status,customer_id (list orders)
  
  Events Published:
    - OrderCreated { orderId, customerId, items, amount }
    - OrderConfirmed { orderId, paymentId }
    - OrderShipped { orderId, trackingNumber }
    - OrderCancelled { orderId, reason }
  
  Events Subscribed:
    - PaymentSucceeded (to transition order to CONFIRMED)
    - PaymentFailed (to keep order in PENDING)
    - InventoryReservationFailed (to cancel order)

Microservice Name: inventory-service
  Bounded Context: Inventory Management
  Responsibilities:
    - Maintain current stock levels
    - Check availability
    - Reserve stock for confirmed orders
    - Release stock for cancelled orders
    - Handle restocking events
  
  Database: inventory_service_db (Oracle)
    Tables: products, stock_movements
    Owned by: inventory-service
  
  APIs (REST):
    GET /api/v1/products/{id}/availability (check stock)
    POST /api/v1/products/{id}/reserve (reserve stock)
    POST /api/v1/products/{id}/release (release reservation)
    GET /api/v1/products/{id}/stock (get current stock)
  
  Events Published:
    - StockReserved { productId, quantity, orderId }
    - StockReservationFailed { productId, quantity, orderId, reason }
    - StockReleased { productId, quantity, orderId }
  
  Events Subscribed:
    - OrderCreated (to process reservations)
    - OrderCancelled (to release reservations)

[Similar patterns for payment-service, notification-service]
```

## Step 4: Identify Data Ownership and Synchronization

Define which service owns which data and how to synchronize:

```
Data Ownership Law:
- Order Service owns ORDERS, ORDER_ITEMS tables
- Inventory Service owns PRODUCTS, STOCK_MOVEMENTS tables
- Payment Service owns PAYMENTS, TRANSACTIONS tables
- NO service (except owner) modifies its data directly

Synchronization Pattern: Event-Driven Eventual Consistency
1. OrderService creates order (PENDING), publishes OrderCreated event
2. InventoryService receives OrderCreated, reserves stock
3. InventoryService publishes StockReserved event
4. PaymentService receives OrderCreated, initiates charge
5. PaymentService publishes PaymentSucceeded event
6. OrderService receives PaymentSucceeded, transitions to CONFIRMED

Result: Order status is source of truth in order-service, stock status in inventory-service
Both converge to consistent state through events (eventual consistency)
```

## Step 5: Define Service-to-Service Communication

Choose synchronous or asynchronous patterns:

```
Pattern 1: Asynchronous Events (Recommended for loose coupling)
When: OrderService publishes OrderCreated event
Who listens: InventoryService, PaymentService, NotificationService
How: Message broker (RabbitMQ, Kafka, or AWS SQS)
Benefits: Resilient, can retry if service down
Drawbacks: Eventual consistency, harder to trace errors

Pattern 2: Synchronous REST Calls (Use sparingly)
When: OrderService needs to check inventory BEFORE creating order
Synchronous call: OrderService → InventoryService.checkAvailability()
Drawback: Blocks order creation on inventory service availability
Alternative: Use async + saga pattern if critical

Pattern 3: Saga Pattern (For distributed transactions)
When: CreateOrder spans multiple services
Saga Orchestrator (OrderService): 
  1. Create order (local)
  2. Call InventoryService.reserve() (if fails, compensate)
  3. Call PaymentService.charge() (if fails, compensate)
  4. Publish OrderConfirmed event
Saga Choreography (Event-driven):
  1. OrderService publishes OrderCreated
  2. InventoryService listens, reserves stock, publishes StockReserved
  3. PaymentService listens, charges, publishes PaymentSucceeded
  4. OrderService listens, confirms order
```

## Step 6: Identify Anti-Corruption Layers

When integrating with external systems, isolate contracts:

```
External PaymentGateway Interface (e.g., Stripe):
  POST /v1/charges { amount, currency, source, description }
  Response: { id, amount, currency, status, created } 

PaymentService Anti-Corruption Layer:
  // Don't let Stripe's contract leak into domain
  StripePaymentGateway implements IPaymentGateway {
    charge(payment: Payment): PaymentResult {
      // Translate Payment domain model → Stripe API request
      stripeRequest = {
        amount: payment.amount.toStripeFormat(),
        currency: payment.currency.code(),
        source: payment.method.stripeTokenId,
        description: "Order " + payment.orderId
      }
      stripeResponse = callStripe(stripeRequest)
      // Translate Stripe response → Payment domain model
      return PaymentResult {
        transactionId: stripeResponse.id,
        status: mapStripeStatus(stripeResponse.status),
        amount: Money.fromCents(stripeResponse.amount, stripeResponse.currency)
      }
    }
  }
```

## Step 7: Create Context Map

Visual diagram showing bounded contexts and their relationships:

```
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY                             │
│          (Routes external requests to services)              │
└────────────┬──────────────────────┬──────────────┬──────────┘
             │                      │              │
       ┌─────▼──────┐        ┌──────▼──────┐ ┌────▼────────┐
       │   Order     │        │ Inventory   │ │ Payment     │
       │  Service    │        │  Service    │ │ Service     │
       │             │        │             │ │             │
       │  owns:      │◄───────│  owns:      │ │ owns:       │
       │  ORDERS     │   data │  PRODUCTS   │ │ PAYMENTS    │
       │  ORDER_     │   sync │  STOCK_     │ │ TRANS-      │
       │  ITEMS      │◄───────│  MOVEMENSTS │ │ ACTIONS     │
       └──────┬──────┘        └──────┬──────┘ │             │
              │ events               │        │  events     │
              │ (OrderCreated,       │        │             │
              │  OrderConfirmed)     │        │             │
              │                      │        │             │
              └──────────┬───────────┼────────┼──────┐      │
                         │           │        │      │      │
                    ┌────▼───────────▼────────▼──────▼──┐  │
                    │   Event Bus (Kafka/RabbitMQ)      │  │
                    │  (Decouples services)             │  │
                    └────────────────┬───────────────────┘  │
                                     │                       │
                              ┌──────▼────────────┐          │
                              │  Notification     │          │
                              │  Service          │          │
                              │                   │          │
                              │  owns: NOTIF-      │          │
                              │  ICATIONS          │          │
                              └────────────────────┘          │
```

## Key Patterns to Apply

### 1. Aggregate Root Pattern
Each bounded context has one or more aggregate roots (e.g., Order, Product, Payment). Treat as transaction boundaries.

### 2. Repository Pattern
Each service has repositories that only load/save its own aggregates.

### 3. Event Sourcing (Optional)
Store all state changes as immutable events, rebuild state by replaying events.

### 4. CQRS (Optional)
Separate read and write models for services that have complex queries.

## Validation Checklist

Before proceeding to implementation:

- [ ] Each bounded context maps to exactly one microservice
- [ ] Each microservice owns exactly one database
- [ ] No two services access the same table
- [ ] All cross-service communication is via REST/events, not shared DB
- [ ] Aggregate boundaries are clear (root + child entities)
- [ ] Ubiquitous language is consistent across team
- [ ] Data synchronization strategy is documented (eventual consistency OK?)
- [ ] External integrations have anti-corruption layers

## Example Deliverable

```markdown
# OrderService System: DDD Service Boundaries

## Bounded Contexts Identified

1. **Order Management** → order-service
   - Core responsibility: Accept and manage customer orders
   - Aggregates: Order (root with OrderItems)
   - Database: ORDERS, ORDER_ITEMS
   - REST APIs: POST/GET/PATCH/DELETE /api/v1/orders

2. **Inventory Management** → inventory-service
   - Core responsibility: Track product availability
   - Aggregates: Product (root with StockMovements)
   - Database: PRODUCTS, STOCK_MOVEMENTS
   - REST APIs: GET /api/v1/products/*/availability, POST /api/v1/products/*/reserve

3. **Payment Processing** → payment-service
   - Core responsibility: Process and track payments
   - Aggregates: Payment (root)
   - Database: PAYMENTS, TRANSACTIONS
   - REST APIs: POST /api/v1/payments, GET /api/v1/payments/*

4. **Notifications** → notification-service
   - Core responsibility: Send messages to customers
   - Aggregates: Notification (root)
   - Database: NOTIFICATIONS, EMAIL_TEMPLATES, SMS_TEMPLATES
   - REST APIs: POST /api/v1/notifications

## Communication Patterns

- Order → Inventory: Events (OrderCreated, OrderCancelled)
- Order → Payment: Events (OrderCreated)
- Payment → Order: Events (PaymentSucceeded, PaymentFailed)
- All → Notification: Events

## Data Synchronization

- Owned: Service owns its tables; no one else modifies
- Eventual consistency: Via event propagation
- Single source of truth: Order status in order-service, Product stock in inventory-service

## Context Map Diagram

[Includes diagram showing all bounded contexts and event flows]
```
