# How to Design MySQL Schema for Microservices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Microservice, Schema Design, Database

Description: Learn how to design MySQL schemas for microservices architectures, covering service boundaries, avoiding shared tables, handling foreign keys across services, and data consistency patterns.

---

## Schema Design Principles for Microservices

In a microservices architecture, each service owns its data. The database schema decisions you make determine how independently services can evolve, deploy, and scale. Poor schema boundaries create tight coupling that undermines the benefits of microservices.

Key principles:
- Each service owns its own MySQL schema (database)
- No service reads directly from another service's tables
- Cross-service relationships are by natural key, not foreign key
- Schema changes in one service must not require changes in another

## Define Service Boundaries

Map services to separate MySQL databases, not just separate tables:

```sql
-- Order Service database
CREATE DATABASE order_service;

-- Customer Service database
CREATE DATABASE customer_service;

-- Inventory Service database
CREATE DATABASE inventory_service;
```

Each service connects to its own database with its own credentials.

## Order Service Schema

```sql
USE order_service;

CREATE TABLE orders (
  id           BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  order_uuid   CHAR(36)      NOT NULL DEFAULT (UUID()),
  customer_id  CHAR(36)      NOT NULL,  -- natural key, NOT a FK to customer_service
  status       VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
  total        DECIMAL(12,2) NOT NULL,
  currency     CHAR(3)       NOT NULL DEFAULT 'USD',
  created_at   TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at   TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uq_uuid (order_uuid),
  INDEX idx_customer (customer_id),
  INDEX idx_status   (status, created_at)
) ENGINE=InnoDB;

CREATE TABLE order_items (
  id          BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  order_id    BIGINT        NOT NULL,
  product_sku VARCHAR(50)   NOT NULL,  -- natural key from inventory_service
  quantity    INT           NOT NULL,
  unit_price  DECIMAL(10,2) NOT NULL,
  CONSTRAINT fk_item_order FOREIGN KEY (order_id) REFERENCES orders(id)
    ON DELETE CASCADE
) ENGINE=InnoDB;
```

Notice `customer_id` uses a UUID string, not an integer foreign key. Cross-service relationships use natural (business) keys.

## Customer Service Schema

```sql
USE customer_service;

CREATE TABLE customers (
  id           BIGINT  NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_uuid CHAR(36) NOT NULL DEFAULT (UUID()),
  email        VARCHAR(200) NOT NULL UNIQUE,
  full_name    VARCHAR(200) NOT NULL,
  tier         VARCHAR(20)  NOT NULL DEFAULT 'STANDARD',
  created_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uq_uuid (customer_uuid)
) ENGINE=InnoDB;
```

## Avoid Cross-Service JOINs

Instead of joining across databases (which creates tight coupling), services communicate via APIs:

```sql
-- BAD: cross-database join couples two services
SELECT o.id, c.email
FROM order_service.orders o
JOIN customer_service.customers c ON c.customer_uuid = o.customer_id;

-- GOOD: Order service queries its own data, fetches customer data via API call
SELECT id, customer_id, total, status
FROM order_service.orders
WHERE customer_id = ?;
-- Application then calls customer-service API to enrich with customer name/email
```

## Denormalize Locally for Read Performance

Store copies of frequently-needed cross-service data locally to avoid API calls on every read:

```sql
USE order_service;

-- Store a snapshot of customer name at order time to avoid API calls
ALTER TABLE orders
  ADD COLUMN customer_name VARCHAR(200) NOT NULL DEFAULT '',
  ADD COLUMN customer_email VARCHAR(200) NOT NULL DEFAULT '';
```

Update this snapshot via events when customer data changes.

## Event-Driven Schema Synchronization

```sql
-- order_service: outbox table for publishing events
CREATE TABLE outbox_events (
  id           BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  event_type   VARCHAR(100) NOT NULL,
  aggregate_id VARCHAR(100) NOT NULL,
  payload      JSON         NOT NULL,
  published    TINYINT(1)   NOT NULL DEFAULT 0,
  created_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_unpublished (published, created_at)
) ENGINE=InnoDB;
```

## Schema Versioning

Each service manages its own migrations independently:

```bash
# order-service migrations
migrations/
  001_create_orders.sql
  002_add_currency.sql
  003_add_customer_snapshot.sql
```

Use tools like Flyway, Liquibase, or golang-migrate per service. Never coordinate migrations across services.

## Summary

MySQL schema design for microservices means giving each service its own database, using natural keys for cross-service references, avoiding cross-database JOINs, and handling consistency through events rather than foreign keys. Denormalize frequently-read cross-service data locally and use an outbox table for reliable event publishing. Each service owns its migration history independently.
