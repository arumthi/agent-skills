# Legacy SOAP/WSDL System Analysis Template

**Project Name**: [Your Project Name]  
**Analysis Date**: [Date]  
**Analyst**: [Your Name]  
**Stakeholders**: [List key stakeholders]  

---

## Executive Summary

Provide a high-level overview of the legacy system, migration scope, and key challenges.

```
LEGACY SYSTEM PROFILE
├── Application Type: Monolithic SOAP/WSDL service
├── Language/Framework: Java [version], Spring 2.x
├── Data Store: Oracle 19c (single schema)
├── User Base: [Number] concurrent users
├── SLAs: [Response time target, availability target]
├── External Integrations: [Number] external systems
└── Estimated Effort: [High/Medium/Low] - roughly [X] person-weeks
```

---

## 1. SOAP/WSDL Service Inventory

List all SOAP services, ports, and operations in the legacy system.

### Service: OrderService
**WSDL URL**: [URL or file path]  
**Namespace**: [XML namespace]  
**Port**: [Port name]

| Operation | Input Message | Output Message | Async? | Load (calls/sec) |
|-----------|---|---|---|---|
| createOrder | CreateOrderRequest | OrderResponse | No | 2-3 |
| getOrder | GetOrderRequest | OrderResponse | No | 100-150 |
| updateOrder | UpdateOrderRequest | OrderResponse | No | 3-5 |
| cancelOrder | CancelOrderRequest | void | Yes | 0.5-1 |

**Key Dependencies**: [List services this service calls]
- PaymentService (SOAP)
- InventoryService (REST API)
- NotificationService (JMS Queue)

### Service: [Additional Services]
[Repeat for each SOAP service]

---

## 2. Data Model Analysis

### Monolithic Database Schema

```sql
-- Current tables (simplified)
ORDERS (ORDER_ID, CUSTOMER_ID, CUSTOMER_NAME, ORDER_DATE, STATUS, TOTAL_AMOUNT, 
        SHIPPING_ADDRESS, PAYMENT_METHOD, PAYMENT_REFERENCE, TRANSACTION_ID, 
        INVENTORY_RESERVED, CREATED_DATE, UPDATED_DATE)

ORDER_ITEMS (ORDER_ITEM_ID, ORDER_ID, PRODUCT_ID, PRODUCT_NAME, QUANTITY, 
             UNIT_PRICE, DISCOUNT_PERCENT, GROSS_TOTAL, CREATED_DATE)

PRODUCTS (PRODUCT_ID, PRODUCT_NAME, DESCRIPTION, UNIT_PRICE, CURRENT_STOCK, 
          RESERVED_STOCK, CREATED_DATE)

PAYMENTS (PAYMENT_ID, ORDER_ID, PAYMENT_METHOD, PAYMENT_REFERENCE, AMOUNT, 
          STATUS, TRANSACTION_ID, CREATED_DATE)

NOTIFICATIONS (NOTIFICATION_ID, CUSTOMER_ID, ORDER_ID, MESSAGE_TYPE, SUBJECT, 
               BODY, SENT_DATE, CREATED_DATE)
```

### Data Volume Estimate

| Table | Rows | Size (GB) | Growth Rate |
|-------|------|-----------|-------------|
| ORDERS | [Estimate] | [Estimate] | [Estimate] |
| ORDER_ITEMS | [Estimate] | [Estimate] | [Estimate] |
| PRODUCTS | [Estimate] | [Estimate] | [Estimate] |
| PAYMENTS | [Estimate] | [Estimate] | [Estimate] |

**Total Database Size**: [Approximate GB]  
**Backup/Restore Time**: [Estimate hours]  

---

## 3. External Integrations

List all external systems this application depends on.

| System | Type | Protocol | Purpose | Criticality |
|--------|------|----------|---------|-------------|
| PaymentGateway | Payment Processor | SOAP | Process card payments | Critical |
| Email Service | Notification | SMTP/REST | Send order confirmations | High |
| Shipping Carrier | Logistics | REST API | Get shipment tracking | Medium |
| Legacy ERP | Enterprise System | File-based FTP | Daily inventory sync | High |

### Integration Challenges
- [ ] Synchronous payment processing blocks order creation
- [ ] Legacy ERP only supports FTP file exchange (no API)
- [ ] Email service has rate limiting (100 msgs/sec)
- [ ] Shipping API requires authentication renewal every 24 hours

---

## 4. Performance Characteristics

### Current Throughput & Load

```
PEAK LOAD PROFILE (Business peak hours)
├── Orders/second: [X]
├── API calls/second: [Y]
├── Database queries/second: [Z]
├── Data transferred/second: [N] MB/s
└── Concurrent connections: [M]

RESPONSE TIME (Current Baseline)
├── createOrder (p50): [X]ms, (p95): [Y]ms, (p99): [Z]ms
├── getOrder (p50): [X]ms, (p95): [Y]ms, (p99): [Z]ms
├── updateOrder (p50): [X]ms, (p95): [Y]ms, (p99): [Z]ms
└── cancelOrder (p50): [X]ms, (p95): [Y]ms, (p99): [Z]ms
```

### Database Performance Issues
- [ ] N+1 query problem in order retrieval
- [ ] Missing indexes on frequently filtered columns
- [ ] Full table scans on order search
- [ ] Slow stored procedures for reconciliation

---

## 5. Identified Business Domains (DDD)

Group WSDL operations by business domain.

### Domain 1: Order Management
**Operations**:
- createOrder
- getOrder
- updateOrder
- cancelOrder
- listOrders

**Data Owned**:
- ORDERS table
- ORDER_ITEMS table
- Order status state machine

**Dependencies**:
- Depends on: PaymentService, InventoryService
- Depended on by: NotificationService

**Target Microservice**: order-service

### Domain 2: Inventory Management
**Operations**:
- checkAvailability
- reserveStock
- releaseStock
- updateInventory

**Data Owned**:
- PRODUCTS table
- STOCK_MOVEMENTS table

**Dependencies**:
- Depends on: Nothing (standalone)
- Depended on by: OrderService

**Target Microservice**: inventory-service

### Domain 3: Payment Processing
**Operations**:
- initiatePayment
- confirmPayment
- refundPayment

**Data Owned**:
- PAYMENTS table
- TRANSACTIONS table

**Dependencies**:
- Depends on: PaymentGateway
- Depended on by: OrderService

**Target Microservice**: payment-service

### Domain 4: Notifications
**Operations**:
- sendOrderConfirmation
- sendShipmentUpdate
- sendPaymentReceipt

**Data Owned**:
- NOTIFICATIONS table
- EMAIL_TEMPLATES table
- SMS_TEMPLATES table

**Dependencies**:
- Depends on: Email Service, SMS Service
- Depended on by: All other services (via events)

**Target Microservice**: notification-service

---

## 6. Integration Points & Patterns

### Current Communication Patterns

```
Order Service
├── createOrder() 
│   ├── Synchronous DB update
│   ├── Synchronous call to PaymentService.charge()
│   ├── If payment succeeds:
│   │   ├── Synchronous call to InventoryService.reserve()
│   │   ├── If inventory reserved: return OrderConfirmed
│   │   └── If inventory fails: rollback payment, return error
│   └── If payment fails: return error without order creation
│
├── updateOrder()
│   └── Synchronous DB update + event publish
│
└── cancelOrder()
    ├── Update DB status to CANCELLED
    ├── Async call to InventoryService.release()
    └── Publish CancelledOrder event to JMS
```

### Migration Challenges
- [ ] **Long-running transactions**: createOrder blocks waiting for payment (30-60 sec timeouts)
- [ ] **Synchronous chain**: Order → Payment → Inventory (cascading failure risk)
- [ ] **No idempotency**: Duplicate createOrder calls create duplicate orders
- [ ] **Implicit contracts**: No API documentation; contracts inferred from WSDL

---

## 7. Identified Risks & Constraints

### Technical Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Long-running transactions block order creation | High | Implement saga pattern for async coordination |
| Payment gateway timeout during migration | High | Add circuit breaker; implement timeout policy |
| Missing indexes slow down order queries | Medium | Profile queries; add indexes during migration |
| No distributed tracing in legacy system | Medium | Implement OpenTelemetry for observability |

### Business Constraints

- [ ] **Backward compatibility required**: Existing SOAP clients must continue working during migration
- [ ] **Zero downtime**: Cannot take system offline; must migrate on live traffic
- [ ] **Data consistency**: Financial transactions must not be lost or duplicated
- [ ] **Regulatory**: [If applicable - e.g., PCI DSS for payment data, GDPR for customer data]

### Dependency Constraints

- [ ] Payment Gateway unavailable for 2 hours every Sunday (maintenance window)
- [ ] Legacy ERP system cannot be modified (owned by different team)
- [ ] Shipping API has rate limit of 1000 reqs/min

---

## 8. Estimated Effort & Timeline

### Migration Phases

| Phase | Duration | Effort | Deliverables |
|-------|----------|--------|--------------|
| Assessment & Design | 2 weeks | 40 hours | DDD service boundaries, architecture docs |
| Build Safety Net | 1 week | 30 hours | Characterization tests, baseline metrics |
| Implement Services | 4 weeks | 120 hours | All microservices, REST APIs, tests |
| OpenShift Setup | 1 week | 20 hours | Helm charts, K8s manifests, CI/CD pipeline |
| E2E Testing | 2 weeks | 60 hours | Playwright test suite, performance tests |
| Strangler Facade | 1 week | 25 hours | API gateway, traffic routing, feature flags |
| Traffic Migration | 2 weeks | 40 hours | 5% → 25% → 50% → 100% phased rollout |
| **TOTAL** | **13 weeks** | **335 hours** | **Production deployment** |

---

## 9. Success Criteria & Acceptance

### Technical Success Criteria

✅ All SOAP operations migrated to REST endpoints  
✅ Database schema decomposed into service-owned databases  
✅ End-to-end tests pass (80%+ coverage)  
✅ Performance baselines met: response time < [X]ms (p99)  
✅ Zero data loss during migration  
✅ 99.9% availability during migration phases  

### Business Success Criteria

✅ 100% of existing SOAP clients' requests routed successfully  
✅ No revenue loss or order fulfillment delays  
✅ All regulatory requirements met  
✅ Operations team trained on new microservices architecture  

---

## 10. Next Steps

1. **Review & Approve**: Stakeholder sign-off on bounded contexts and service architecture
2. **Create Detailed ADRs**: Document architectural decisions
3. **Provision Infrastructure**: Set up OpenShift namespace, databases, logging stack
4. **Start Implementation**: Begin with highest-priority microservice (e.g., order-service)
5. **Build Automation**: CI/CD pipeline, test infrastructure, monitoring
6. **Plan Cutover**: Create detailed runbooks for traffic migration phases

---

## Appendix: WSDL Schema Examples

```xml
<!-- Example WSDL message definitions -->
<wsdl:message name="CreateOrderRequest">
  <wsdl:part name="body" type="xsd:Order"/>
</wsdl:message>

<wsdl:message name="OrderResponse">
  <wsdl:part name="body" type="xsd:OrderResult"/>
</wsdl:message>

<!-- XSD Type Definitions -->
<xsd:complexType name="Order">
  <xsd:sequence>
    <xsd:element name="customerId" type="xsd:string"/>
    <xsd:element name="items" type="OrderItem" maxOccurs="unbounded"/>
    <xsd:element name="shippingAddress" type="Address"/>
  </xsd:sequence>
</xsd:complexType>
```

---

## Document Control

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.0 | [Date] | [Name] | Initial analysis |
| 1.1 | [Date] | [Name] | [Change description] |
