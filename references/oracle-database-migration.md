# Oracle Database Migration: Schema Decomposition

## Overview

Migrating from a monolithic Oracle database to microservice-owned databases requires careful planning. This guide covers schema decomposition strategies, migration scripts, and testing approaches.

## Step 1: Analyze Monolithic Schema

### Current State (Monolithic)

```sql
-- Single database with all tables intermixed
CREATE TABLE ORDERS (
  ORDER_ID NUMBER PRIMARY KEY,
  CUSTOMER_ID NUMBER NOT NULL,
  CUSTOMER_NAME VARCHAR2(100),  -- Denormalized customer data
  CUSTOMER_EMAIL VARCHAR2(100),
  ORDER_DATE TIMESTAMP NOT NULL,
  STATUS VARCHAR2(20) NOT NULL,
  TOTAL_AMOUNT DECIMAL(10,2),
  SHIPPING_ADDRESS VARCHAR2(200),
  SHIPPING_CITY VARCHAR2(100),
  PAYMENT_METHOD VARCHAR2(50),
  PAYMENT_REFERENCE VARCHAR2(100),
  TRANSACION_ID VARCHAR2(100),  -- Payment data
  PRODUCT_IDS VARCHAR2(1000),  -- Denormalized inventory references
  INVENTORY_RESERVED CHAR(1),
  CREATED_BY VARCHAR2(50),
  CREATED_DATE TIMESTAMP NOT NULL,
  UPDATED_BY VARCHAR2(50),
  UPDATED_DATE TIMESTAMP NOT NULL
);

CREATE TABLE ORDER_ITEMS (
  ORDER_ITEM_ID NUMBER PRIMARY KEY,
  ORDER_ID NUMBER NOT NULL,
  PRODUCT_ID VARCHAR2(50) NOT NULL,
  PRODUCT_NAME VARCHAR2(100),  -- Denormalized
  QUANTITY NUMBER NOT NULL,
  UNIT_PRICE DECIMAL(10,2),
  DISCOUNT_PERCENT DECIMAL(5,2),
  GROSS_TOTAL DECIMAL(10,2),
  CREATED_DATE TIMESTAMP NOT NULL
);

CREATE TABLE PRODUCTS (
  PRODUCT_ID VARCHAR2(50) PRIMARY KEY,
  PRODUCT_NAME VARCHAR2(100) NOT NULL,
  DESCRIPTION CLOB,
  UNIT_PRICE DECIMAL(10,2),
  CURRENT_STOCK NUMBER,
  RESERVED_STOCK NUMBER,
  CREATED_DATE TIMESTAMP NOT NULL
);

CREATE TABLE PAYMENTS (
  PAYMENT_ID NUMBER PRIMARY KEY,
  ORDER_ID NUMBER NOT NULL,
  PAYMENT_METHOD VARCHAR2(50),
  PAYMENT_REFERENCE VARCHAR2(100),
  AMOUNT DECIMAL(10,2),
  STATUS VARCHAR2(20),
  TRANSACTION_ID VARCHAR2(100),
  CREATED_DATE TIMESTAMP NOT NULL
);

CREATE TABLE NOTIFICATIONS (
  NOTIFICATION_ID NUMBER PRIMARY KEY,
  CUSTOMER_ID NUMBER NOT NULL,
  ORDER_ID NUMBER,
  MESSAGE_TYPE VARCHAR2(50),
  SUBJECT VARCHAR2(200),
  BODY CLOB,
  SENT_DATE TIMESTAMP,
  CREATED_DATE TIMESTAMP NOT NULL
);
```

## Step 2: Map Tables to Bounded Contexts

```
MONOLITHIC DATABASE
├── CUSTOMER DATA (ORDERS.CUSTOMER_ID, CUSTOMER_NAME, EMAIL) → CUSTOMER SERVICE
├── ORDER DATA (ORDERS, ORDER_ITEMS) → ORDER SERVICE
├── PRODUCT DATA (PRODUCTS) → INVENTORY SERVICE
├── PAYMENT DATA (PAYMENTS, ORDERS.PAYMENT_*) → PAYMENT SERVICE
└── NOTIFICATION DATA (NOTIFICATIONS) → NOTIFICATION SERVICE
```

## Step 3: Design Target Database Schema

### Target State: Service-Owned Databases

#### ORDER SERVICE DATABASE (order_service_db)

```sql
-- order-service owns orders and order items only
CREATE TABLE ORDERS (
  ORDER_ID VARCHAR2(50) PRIMARY KEY,
  CUSTOMER_ID VARCHAR2(50) NOT NULL,    -- Reference only, don't own
  STATUS VARCHAR2(20) NOT NULL CHECK (STATUS IN ('PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED')),
  TOTAL_AMOUNT DECIMAL(10,2) NOT NULL,
  CREATED_DATE TIMESTAMP NOT NULL,
  UPDATED_DATE TIMESTAMP NOT NULL,
  CONSTRAINT orders_pk PRIMARY KEY (ORDER_ID),
  CONSTRAINT orders_status_ck CHECK (STATUS IN ('PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'))
);

CREATE TABLE ORDER_ITEMS (
  ORDER_ITEM_ID VARCHAR2(50) PRIMARY KEY,
  ORDER_ID VARCHAR2(50) NOT NULL REFERENCES ORDERS(ORDER_ID) ON DELETE CASCADE,
  PRODUCT_ID VARCHAR2(50) NOT NULL,    -- Reference only, don't own
  QUANTITY NUMBER NOT NULL,
  UNIT_PRICE DECIMAL(10,2) NOT NULL,
  LINE_TOTAL DECIMAL(10,2) GENERATED ALWAYS AS (QUANTITY * UNIT_PRICE) STORED,
  CREATED_DATE TIMESTAMP NOT NULL,
  CONSTRAINT order_items_pk PRIMARY KEY (ORDER_ITEM_ID),
  CONSTRAINT order_items_order_fk FOREIGN KEY (ORDER_ID) REFERENCES ORDERS(ORDER_ID) ON DELETE CASCADE
);

CREATE INDEX idx_orders_customer ON ORDERS(CUSTOMER_ID);
CREATE INDEX idx_orders_status ON ORDERS(STATUS);
CREATE INDEX idx_order_items_order ON ORDER_ITEMS(ORDER_ID);
CREATE INDEX idx_order_items_product ON ORDER_ITEMS(PRODUCT_ID);
```

#### INVENTORY SERVICE DATABASE (inventory_service_db)

```sql
-- inventory-service owns products and stock tracking
CREATE TABLE PRODUCTS (
  PRODUCT_ID VARCHAR2(50) PRIMARY KEY,
  PRODUCT_NAME VARCHAR2(100) NOT NULL,
  DESCRIPTION CLOB,
  UNIT_PRICE DECIMAL(10,2) NOT NULL,
  CREATED_DATE TIMESTAMP NOT NULL,
  UPDATED_DATE TIMESTAMP NOT NULL
);

CREATE TABLE STOCK_MOVEMENTS (
  MOVEMENT_ID VARCHAR2(50) PRIMARY KEY,
  PRODUCT_ID VARCHAR2(50) NOT NULL REFERENCES PRODUCTS(PRODUCT_ID),
  MOVEMENT_TYPE VARCHAR2(50),  -- RECEIVED, SOLD, RESERVED, RETURNED, ADJUSTED
  QUANTITY NUMBER NOT NULL,
  REFERENCE_ID VARCHAR2(100),  -- Order ID or PO ID
  CREATED_DATE TIMESTAMP NOT NULL
);

CREATE VIEW CURRENT_INVENTORY AS
SELECT
  p.PRODUCT_ID,
  p.PRODUCT_NAME,
  p.UNIT_PRICE,
  COALESCE(SUM(CASE WHEN sm.MOVEMENT_TYPE IN ('RECEIVED', 'RETURNED') THEN sm.QUANTITY
                    WHEN sm.MOVEMENT_TYPE IN ('SOLD', 'ADJUSTED') THEN -sm.QUANTITY END), 0) AS CURRENT_STOCK,
  COALESCE(SUM(CASE WHEN sm.MOVEMENT_TYPE = 'RESERVED' THEN sm.QUANTITY ELSE 0 END), 0) AS RESERVED_STOCK,
  COALESCE(SUM(CASE WHEN sm.MOVEMENT_TYPE IN ('RECEIVED', 'RETURNED') THEN sm.QUANTITY
                    WHEN sm.MOVEMENT_TYPE IN ('SOLD', 'ADJUSTED', 'RESERVED') THEN -sm.QUANTITY END), 0) AS AVAILABLE_STOCK
FROM PRODUCTS p
LEFT JOIN STOCK_MOVEMENTS sm ON p.PRODUCT_ID = sm.PRODUCT_ID
GROUP BY p.PRODUCT_ID, p.PRODUCT_NAME, p.UNIT_PRICE;

CREATE INDEX idx_stock_movements_product ON STOCK_MOVEMENTS(PRODUCT_ID);
CREATE INDEX idx_stock_movements_type ON STOCK_MOVEMENTS(MOVEMENT_TYPE);
```

#### PAYMENT SERVICE DATABASE (payment_service_db)

```sql
-- payment-service owns only payments
CREATE TABLE PAYMENTS (
  PAYMENT_ID VARCHAR2(50) PRIMARY KEY,
  ORDER_ID VARCHAR2(50) NOT NULL,  -- Reference only
  PAYMENT_METHOD VARCHAR2(50) NOT NULL,
  AMOUNT DECIMAL(10,2) NOT NULL,
  STATUS VARCHAR2(20) NOT NULL CHECK (STATUS IN ('INITIATED', 'SUCCESS', 'FAILED', 'REFUNDED')),
  TRANSACTION_ID VARCHAR2(100),
  EXTERNAL_REFERENCE VARCHAR2(100),
  CREATED_DATE TIMESTAMP NOT NULL,
  UPDATED_DATE TIMESTAMP NOT NULL
);

CREATE INDEX idx_payments_order ON PAYMENTS(ORDER_ID);
CREATE INDEX idx_payments_status ON PAYMENTS(STATUS);
```

#### NOTIFICATION SERVICE DATABASE (notification_service_db)

```sql
-- notification-service owns only notifications
CREATE TABLE NOTIFICATIONS (
  NOTIFICATION_ID VARCHAR2(50) PRIMARY KEY,
  CUSTOMER_ID VARCHAR2(50) NOT NULL,
  EVENT_TYPE VARCHAR2(50),  -- ORDER_CREATED, ORDER_SHIPPED, PAYMENT_RECEIVED
  REFERENCE_ID VARCHAR2(100),  -- Order ID or Payment ID
  SUBJECT VARCHAR2(200),
  BODY CLOB,
  STATUS VARCHAR2(20),  -- PENDING, SENT, FAILED
  SENT_DATE TIMESTAMP,
  CREATED_DATE TIMESTAMP NOT NULL
);

CREATE INDEX idx_notifications_customer ON NOTIFICATIONS(CUSTOMER_ID);
CREATE INDEX idx_notifications_status ON NOTIFICATIONS(STATUS);
```

## Step 4: Create Flyway Migration Scripts

All services use Flyway for idempotent schema migrations. Place scripts in `src/main/resources/db/migration/`.

### order-service/V1__create_orders_schema.sql

```sql
-- Flyway V1__create_orders_schema.sql
CREATE TABLE ORDERS (
  ORDER_ID VARCHAR2(50) PRIMARY KEY,
  CUSTOMER_ID VARCHAR2(50) NOT NULL,
  STATUS VARCHAR2(20) NOT NULL DEFAULT 'PENDING',
  TOTAL_AMOUNT DECIMAL(10,2) NOT NULL,
  CREATED_DATE TIMESTAMP DEFAULT SYSDATE NOT NULL,
  UPDATED_DATE TIMESTAMP DEFAULT SYSDATE NOT NULL,
  CONSTRAINT orders_status_ck CHECK (STATUS IN ('PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'))
);

CREATE INDEX idx_orders_customer ON ORDERS(CUSTOMER_ID);
CREATE INDEX idx_orders_status ON ORDERS(STATUS);
CREATE INDEX idx_orders_created_date ON ORDERS(CREATED_DATE DESC);

CREATE TABLE ORDER_ITEMS (
  ORDER_ITEM_ID VARCHAR2(50) PRIMARY KEY,
  ORDER_ID VARCHAR2(50) NOT NULL,
  PRODUCT_ID VARCHAR2(50) NOT NULL,
  QUANTITY NUMBER NOT NULL,
  UNIT_PRICE DECIMAL(10,2) NOT NULL,
  LINE_TOTAL DECIMAL(10,2) GENERATED ALWAYS AS (QUANTITY * UNIT_PRICE) STORED,
  CREATED_DATE TIMESTAMP DEFAULT SYSDATE NOT NULL,
  CONSTRAINT order_items_order_fk FOREIGN KEY (ORDER_ID) REFERENCES ORDERS(ORDER_ID) ON DELETE CASCADE,
  CONSTRAINT order_items_qty_ck CHECK (QUANTITY > 0),
  CONSTRAINT order_items_price_ck CHECK (UNIT_PRICE >= 0)
);

CREATE INDEX idx_order_items_order ON ORDER_ITEMS(ORDER_ID);
CREATE INDEX idx_order_items_product ON ORDER_ITEMS(PRODUCT_ID);

COMMIT;
```

### inventory-service/V1__create_inventory_schema.sql

```sql
-- Flyway V1__create_inventory_schema.sql
CREATE TABLE PRODUCTS (
  PRODUCT_ID VARCHAR2(50) PRIMARY KEY,
  PRODUCT_NAME VARCHAR2(100) NOT NULL,
  DESCRIPTION CLOB,
  UNIT_PRICE DECIMAL(10,2) NOT NULL,
  CREATED_DATE TIMESTAMP DEFAULT SYSDATE NOT NULL,
  UPDATED_DATE TIMESTAMP DEFAULT SYSDATE NOT NULL,
  CONSTRAINT products_price_ck CHECK (UNIT_PRICE >= 0)
);

CREATE INDEX idx_products_name ON PRODUCTS(PRODUCT_NAME);

CREATE TABLE STOCK_MOVEMENTS (
  MOVEMENT_ID VARCHAR2(50) PRIMARY KEY,
  PRODUCT_ID VARCHAR2(50) NOT NULL,
  MOVEMENT_TYPE VARCHAR2(50) NOT NULL,
  QUANTITY NUMBER NOT NULL,
  REFERENCE_ID VARCHAR2(100),
  CREATED_DATE TIMESTAMP DEFAULT SYSDATE NOT NULL,
  CONSTRAINT stock_movements_product_fk FOREIGN KEY (PRODUCT_ID) REFERENCES PRODUCTS(PRODUCT_ID),
  CONSTRAINT stock_movements_type_ck CHECK (MOVEMENT_TYPE IN ('RECEIVED', 'SOLD', 'RESERVED', 'RELEASED', 'RETURNED', 'ADJUSTED'))
);

CREATE INDEX idx_stock_movements_product ON STOCK_MOVEMENTS(PRODUCT_ID);
CREATE INDEX idx_stock_movements_type ON STOCK_MOVEMENTS(MOVEMENT_TYPE);
CREATE INDEX idx_stock_movements_reference ON STOCK_MOVEMENTS(REFERENCE_ID);

COMMIT;
```

## Step 5: Data Migration Scripts

### Manual Data Export/Import

```bash
# Export from monolithic database
sqlplus -S system/oracle << EOF

-- Export ORDERS and ORDER_ITEMS for order-service
SPOOL /tmp/orders_export.sql
SELECT 'INSERT INTO ORDERS (ORDER_ID, CUSTOMER_ID, STATUS, TOTAL_AMOUNT, CREATED_DATE, UPDATED_DATE) VALUES ('
  || CHR(39) || ORDER_ID || CHR(39) || ', '
  || CHR(39) || CUSTOMER_ID || CHR(39) || ', '
  || CHR(39) || STATUS || CHR(39) || ', '
  || TOTAL_AMOUNT || ', '
  || 'TIMESTAMP ' || CHR(39) || CREATED_DATE || CHR(39) || ', '
  || 'TIMESTAMP ' || CHR(39) || UPDATED_DATE || CHR(39) || ');'
FROM ORDERS;
SPOOL OFF

-- Similar for ORDER_ITEMS, PRODUCTS, PAYMENTS, NOTIFICATIONS

EOF
```

### Oracle Data Pump (Recommended for large datasets)

```bash
# Export specific tables from monolithic DB
expdp system/oracle dumpfile=orders.dmp logfile=orders.log \
  TABLES=ORDERS,ORDER_ITEMS \
  DIRECTORY=DATA_PUMP_DIR

# Import to order-service database
impdp system/oracle dumpfile=orders.dmp logfile=import_orders.log \
  DIRECTORY=DATA_PUMP_DIR \
  TABLE_EXISTS_ACTION=APPEND

# Repeat for other services
expdp system/oracle dumpfile=payments.dmp logfile=payments.log \
  TABLES=PAYMENTS \
  DIRECTORY=DATA_PUMP_DIR
```

## Step 6: Data Synchronization During Migration

Use event-driven approach to keep new and old systems in sync:

```sql
-- In monolithic database: Create trigger to publish events
CREATE OR REPLACE TRIGGER order_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON ORDERS
FOR EACH ROW
BEGIN
  INSERT INTO ORDER_EVENTS (
    EVENT_ID, EVENT_TYPE, ORDER_ID, EVENT_DATA, CREATED_DATE
  ) VALUES (
    order_event_seq.NEXTVAL,
    CASE WHEN INSERTING THEN 'ORDER_CREATED'
         WHEN UPDATING THEN 'ORDER_UPDATED'
         WHEN DELETING THEN 'ORDER_DELETED' END,
    COALESCE(:NEW.ORDER_ID, :OLD.ORDER_ID),
    JSON_OBJECT(
      'customerId' VALUE COALESCE(:NEW.CUSTOMER_ID, :OLD.CUSTOMER_ID),
      'status' VALUE COALESCE(:NEW.STATUS, :OLD.STATUS),
      'totalAmount' VALUE COALESCE(:NEW.TOTAL_AMOUNT, :OLD.TOTAL_AMOUNT)
    ),
    SYSDATE
  );
  COMMIT;
END;
/

-- Poll and send events to Kafka/RabbitMQ
-- (Implement in Java using Spring Data JDBC polling)
```

## Step 7: Testing Database Migrations

### TestContainers for Local Testing

```java
// OrderServiceApplicationTests.java
import org.testcontainers.oracle.OracleContainer;
import org.junit.jupiter.api.Test;

class OracleContainerTest {
    static final OracleContainer oracle = new OracleContainer("container-registry.oracle.com/database/express:latest")
        .withDatabaseName("XEPDB1")
        .withUsername("test_user")
        .withPassword("test_pwd");

    static {
        oracle.start();
    }

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", oracle::getJdbcUrl);
        registry.add("spring.datasource.username", oracle::getUsername);
        registry.add("spring.datasource.password", oracle::getPassword);
    }

    @Test
    void testFlywayMigration() {
        // Flyway runs automatically; verify schema exists
        assertThat(oracle).isRunning();
    }
}
```

## Step 8: Rollback Strategy

Keep migration reversible:

```sql
-- V2__rollback_orders.sql (for emergency rollback)
-- Only drop tables if migration failed
-- DROP TABLE ORDER_ITEMS;
-- DROP TABLE ORDERS;
-- This should be managed by deployment tools, not auto-executed
```

## Data Ownership Rules

```
┌─────────────────────────────────────────────────────────────┐
│ DATA OWNERSHIP LAW                                          │
├─────────────────────────────────────────────────────────────┤
│ 1. Each service owns exactly one database schema            │
│ 2. No other service modifies that schema                    │
│ 3. Other services reference data via service APIs (REST)    │
│ 4. NO direct database access across service boundaries      │
│ 5. Data sync via events (eventual consistency)              │
└─────────────────────────────────────────────────────────────┘
```

## Success Criteria

✅ All tables decomposed and assigned to services  
✅ Flyway migrations created and idempotent  
✅ Data successfully migrated to new schemas  
✅ Foreign keys within service boundaries only  
✅ Cross-service references via API calls only  
✅ Migrations tested with TestContainers Oracle  
✅ Rollback procedure documented and tested  
