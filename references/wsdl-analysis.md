# WSDL Analysis Guide

## Overview

When migrating from SOAP/WSDL to REST microservices, you must first thoroughly understand the legacy system. This guide helps you extract service contracts, data flows, and dependencies from WSDL files.

## Step 1: Extract WSDL Operations

### What to Document

For each WSDL service and port:

```
Service Name: OrderService
Port: OrderServicePort
Namespace: http://example.com/orders

Operation: createOrder
  Input: CreateOrderRequest
    - customerId: string
    - items: OrderItem[] (contains: productId, quantity, price)
    - shippingAddress: Address
  Output: OrderResponse
    - orderId: string
    - status: enum (PENDING, CONFIRMED, SHIPPED)
    - totalAmount: decimal
  Faults:
    - InvalidOrderException (when items empty or price negative)
    - CustomerNotFoundException (when customerId not found)
    - InsufficientInventoryException (when stock unavailable)

Operation: getOrder
  Input: GetOrderRequest { orderId: string }
  Output: OrderResponse { orderId, status, items, totalAmount }
  Faults: OrderNotFoundException

Operation: updateOrder
  Input: UpdateOrderRequest { orderId, status }
  Output: void
  Faults: OrderNotFoundException, InvalidStateTransitionException

Operation: cancelOrder
  Input: CancelOrderRequest { orderId, reason: string }
  Output: void
  Faults: OrderNotFoundException, CannotCancelException
```

### Tools to Extract WSDL

```bash
# Using SoapUI
1. Open SoapUI
2. File > New SOAP Project
3. Enter WSDL URL/path
4. Inspect each service to document operations

# Using command-line tools
wsimport -keep -p com.generated service.wsdl
# Then inspect generated Java stubs for operation signatures

# Using documentation
grep -n "<wsdl:operation" service.wsdl
grep -n "<xsd:element" service.wsdl
```

## Step 2: Map Data Types and Messages

### Create Type Mapping Document

```
Message: CreateOrderRequest
  Fields:
    customerId: xsd:string (minLength=1, maxLength=20)
    items: OrderItemList (minOccurs=1)
      - productId: xsd:string
      - quantity: xsd:int (minInclusive=1)
      - price: xsd:decimal (fractionDigits=2)
    shippingAddress: Address (nillable=false)
    metadata: optional string

Message: OrderResponse
  Fields:
    orderId: xsd:string (pattern="ORD-[0-9]{10}")
    status: OrderStatus enum (PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED)
    items: OrderItemList
    totalAmount: xsd:decimal (fractionDigits=2)
    createdDate: xsd:dateTime
    updatedDate: xsd:dateTime

Message: OrderStatus
  Enum Values: PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
```

## Step 3: Document Fault Handling

Create a fault matrix:

```
| Operation | Exception Type | HTTP Status (Target) | Error Code | Message |
|-----------|---|---|---|---|
| createOrder | InvalidOrderException | 400 | ORD001 | Order validation failed |
| createOrder | CustomerNotFoundException | 404 | CUST001 | Customer not found |
| createOrder | InsufficientInventoryException | 409 | INV001 | Insufficient inventory |
| getOrder | OrderNotFoundException | 404 | ORD002 | Order not found |
| updateOrder | OrderNotFoundException | 404 | ORD002 | Order not found |
| updateOrder | InvalidStateTransitionException | 400 | ORD003 | Invalid status transition |
| cancelOrder | CannotCancelException | 403 | ORD004 | Order cannot be cancelled |
```

## Step 4: Identify External Dependencies

Document all calls to other systems:

```
OrderService Dependencies:
├── CustomerService
│   └── SOAP operations: getCustomer(customerId) → CustomerDetails
├── InventoryService
│   └── SOAP operations: checkStock(productId, qty) → boolean
│   └── SOAP operations: reserveStock(productId, qty) → ReservationId
├── PaymentService (external REST API)
│   └── POST /payments { orderId, amount, customerId } → { transactionId, status }
├── NotificationService (Message Queue - JMS)
│   └── Queue: order.created, order.shipped, order.cancelled
└── Oracle Database
    └── Tables: ORDERS, ORDER_ITEMS, ORDER_HISTORY
```

## Step 5: Analyze Database Schema

Query the Oracle database to understand data ownership:

```sql
-- Find tables related to WSDL service
SELECT table_name FROM user_tables 
WHERE table_name LIKE '%ORDER%' OR table_name LIKE '%CUSTOMER%';

-- Analyze table structure
DESCRIBE ORDERS;
/*
 Name                  Null?    Type
 ----------------------- -------- ---
 ORDER_ID              NOT NULL NUMBER
 CUSTOMER_ID              NOT NULL NUMBER
 ORDER_DATE            NOT NULL DATE
 STATUS                NOT NULL VARCHAR2(20)
 TOTAL_AMOUNT          NOT NULL DECIMAL(10,2)
 CREATED_BY               NOT NULL VARCHAR2(50)
 CREATED_DATE          NOT NULL TIMESTAMP
 UPDATED_BY               NOT NULL VARCHAR2(50)
 UPDATED_DATE          NOT NULL TIMESTAMP
*/

-- Check constraints and triggers
SELECT constraint_name, constraint_type
FROM user_constraints WHERE table_name = 'ORDERS';

SELECT trigger_name, trigger_type
FROM user_triggers WHERE table_name = 'ORDERS';

-- Find stored procedures invoked by WSDL
SELECT object_name FROM user_objects 
WHERE object_type IN ('PROCEDURE', 'FUNCTION', 'PACKAGE')
AND object_name LIKE '%ORDER%';
```

## Step 6: Document Integration Patterns

Identify how legacy SOAP services communicate:

```
Pattern 1: Request-Response (Synchronous)
  Client → OrderService.createOrder() → OrderService calls PaymentService.charge()
    → PaymentService calls InventoryService.reserveStock()
    → Returns OrderResponse to client

Pattern 2: Fire-and-Forget (Asynchronous)
  OrderService.createOrder() → Enqueues message to JMS topic "order.created"
  NotificationService listens → Sends email/SMS

Pattern 3: Polling
  Client → OrderService.getOrder() (periodically checks status)
  OrderService queries database for status updates

Pattern 4: Stored Procedures
  OrderService operation invokes PL/SQL: ORDERS_PKG.CREATE_ORDER_SP()
```

## Step 7: Create WSDL to REST Mapping

Map each SOAP operation to REST endpoints:

```
SOAP Operation → REST Endpoint

createOrder() → POST /api/v1/orders
  Input: CreateOrderRequest → JSON body
  Output: OrderResponse → 201 Created
  Faults:
    InvalidOrderException → 400 Bad Request
    CustomerNotFoundException → 404 Not Found
    InsufficientInventoryException → 409 Conflict

getOrder(orderId) → GET /api/v1/orders/{orderId}
  Input: orderId path param
  Output: OrderResponse → 200 OK
  Faults:
    OrderNotFoundException → 404 Not Found

updateOrder(orderId, status) → PATCH /api/v1/orders/{orderId}
  Input: JSON { "status": "CONFIRMED" }
  Output: 204 No Content (or OrderResponse 200 OK)
  Faults:
    OrderNotFoundException → 404 Not Found
    InvalidStateTransitionException → 400 Bad Request

cancelOrder(orderId) → DELETE /api/v1/orders/{orderId}
  Input: orderId path param
  Output: 204 No Content
  Faults:
    OrderNotFoundException → 404 Not Found
    CannotCancelException → 403 Forbidden
```

## Step 8: Performance and Load Characteristics

Analyze current usage patterns:

```sql
-- Check operation call frequencies (if logged in database)
SELECT operation_name, COUNT(*) as call_count, AVG(execution_time_ms) as avg_time
FROM soap_audit_log
WHERE created_date > TRUNC(SYSDATE) - 7
GROUP BY operation_name
ORDER BY call_count DESC;

-- Results might show:
operation_name          call_count  avg_time
createOrder            150         245
getOrder               8500        32
updateOrder             250        78
cancelOrder             50          125
```

This tells you:
- `getOrder` is the most frequently called (optimize this first)
- `createOrder` is slower (may need async processing)
- Set performance SLAs: getOrder < 100ms, createOrder < 500ms

## Deliverables

Create a document with these sections:

1. **Executive Summary**
   - Number of SOAP services and operations
   - Main business domains
   - Key risks for migration

2. **WSDL Operations Catalog**
   - All services, ports, operations
   - Input/output message definitions
   - Fault definitions

3. **Database Schema Map**
   - Tables, relationships, triggers, stored procedures
   - Data ownership per business domain
   - Identified constraints

4. **Dependency Graph**
   - Services that depend on this system
   - External systems this depends on
   - Synchronous vs asynchronous patterns

5. **WSDL to REST Mapping**
   - Table of operation → endpoint conversions
   - HTTP verb, path, status codes
   - Error handling strategy

6. **Performance Baseline**
   - Current response times (p50, p95, p99)
   - Peak load (operations per second)
   - Slowest and fastest operations

7. **Migration Risks**
   - Complex state machines in SOAP operations
   - Long-running transactions
   - External integrations that must be preserved
   - Data integrity constraints

## Example Output

```markdown
# OrderService WSDL Analysis

## Services Summary
- ServiceName: OrderService
- Operations: 4 (createOrder, getOrder, updateOrder, cancelOrder)
- DataStore: Oracle 19c (ORDERS, ORDER_ITEMS tables)
- ExternalDeps: PaymentService (REST), InventoryService (SOAP), NotificationService (JMS)

## Operations Mapped to REST

| SOAP Op | REST Endpoint | Method | Sync? | Load (calls/sec) |
|---------|---|---|---|---|
| createOrder | /orders | POST | Yes (blocks on inventory) | 2-3 |
| getOrder | /orders/{id} | GET | Yes | 100-150 |
| updateOrder | /orders/{id} | PATCH | Yes | 3-5 |
| cancelOrder | /orders/{id} | DELETE | Async (JMS) | 0.5-1 |

## Risks
- createOrder synchronously waits for InventoryService response (timeout risk)
- Status field uses custom enum (PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED)
- No idempotency key; duplicate createOrder calls create duplicate orders

## Recommendations
- Implement idempotency keys on createOrder
- Make InventoryService calls async via Saga pattern
- Decompose into UserService, OrderService, InventoryService, PaymentService
```
