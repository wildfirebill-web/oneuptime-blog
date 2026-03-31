# How to Implement Shared Database Pattern with MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Shared Database, Microservice, Pattern

Description: Learn how to implement the shared database pattern with MySQL in microservices, including schema isolation strategies, access control, and migration coordination.

---

## What Is the Shared Database Pattern

The shared database pattern places multiple microservices in the same MySQL instance or even the same database. Unlike the database-per-service pattern, services share the physical database but should still maintain logical isolation through separate schemas, dedicated users, and strict access controls.

This pattern is chosen when:
- Operational complexity of managing many separate MySQL instances is too high
- Cross-service transactions are required (though this is a design smell)
- Teams are small and service boundaries are still evolving

## Option 1 - Separate Schemas, Shared Instance

Each service gets its own MySQL database (schema) on the same server:

```sql
-- All services on the same MySQL instance
CREATE DATABASE order_svc   CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE payment_svc CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE user_svc    CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Each service has a user limited to its own schema
CREATE USER 'order_app'@'%'   IDENTIFIED BY 'pass1';
CREATE USER 'payment_app'@'%' IDENTIFIED BY 'pass2';
CREATE USER 'user_app'@'%'    IDENTIFIED BY 'pass3';

GRANT SELECT, INSERT, UPDATE, DELETE ON order_svc.*   TO 'order_app'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON payment_svc.* TO 'payment_app'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON user_svc.*    TO 'user_app'@'%';
```

This is the recommended form of the shared database pattern because each service still has logical ownership of its data.

## Option 2 - Table Prefix Isolation

When all services share a single database, use table prefixes to namespace ownership:

```sql
-- Naming convention: <service>_<table>
CREATE TABLE order_orders (
  id          BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  user_id     BIGINT        NOT NULL,
  total       DECIMAL(12,2) NOT NULL,
  status      VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
  created_at  TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE payment_payments (
  id          BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  order_id    BIGINT        NOT NULL,
  amount      DECIMAL(12,2) NOT NULL,
  status      VARCHAR(20)   NOT NULL DEFAULT 'INITIATED',
  processed_at TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE user_users (
  id          BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  email       VARCHAR(200) NOT NULL UNIQUE,
  full_name   VARCHAR(200) NOT NULL,
  created_at  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

## Access Control with Views

Restrict cross-service table access by granting view access instead of direct table access:

```sql
-- Allow payment_svc to read a limited view of order data
CREATE VIEW payment_svc.order_totals AS
SELECT id, total, status
FROM order_svc.order_orders
WHERE status IN ('PENDING', 'CONFIRMED');

GRANT SELECT ON payment_svc.order_totals TO 'payment_app'@'%';
```

The payment service sees only the columns and rows it legitimately needs.

## Migration Coordination

With shared databases, teams must coordinate schema changes to avoid breaking other services:

```bash
# Enforce naming in migration files
migrations/
  order-svc/
    001_create_order_orders.sql
    002_add_currency_column.sql
  payment-svc/
    001_create_payment_payments.sql
  user-svc/
    001_create_user_users.sql
```

Use a deployment gate that requires the owning team to approve any migration that touches tables owned by another service.

## Avoiding Cross-Service Joins

Even in a shared database, direct cross-service joins create coupling:

```sql
-- BAD: joins across service boundaries directly
SELECT u.email, o.total
FROM user_users u
JOIN order_orders o ON o.user_id = u.id
WHERE o.status = 'PENDING';

-- GOOD: order service fetches its own data; user data fetched via API
SELECT id, user_id, total
FROM order_orders
WHERE status = 'PENDING';
-- Application calls user-service API for user emails
```

## When to Avoid the Shared Database Pattern

```text
- When services need to scale independently at the database level
- When different services have different storage or engine requirements
- When teams want fully autonomous deployments
- When strict data isolation is required for compliance (PII, PCI)
```

In these cases, migrate toward the database-per-service pattern.

## Example: Migration Path to Database per Service

```sql
-- Step 1: Add a UUID column to the shared table
ALTER TABLE order_orders ADD COLUMN order_uuid CHAR(36) DEFAULT (UUID());

-- Step 2: New service starts reading/writing to its own database
-- using the UUID as the cross-service reference

-- Step 3: Gradually move traffic to the new service's database
-- Step 4: Remove the old tables from the shared database
```

## Summary

The shared database pattern with MySQL works best when each service uses a separate MySQL database on the same server, with dedicated users and strict GRANT controls. Use views to expose limited cross-service read access without coupling schemas. Even in a shared database, avoid direct cross-service joins, coordinate migrations carefully, and treat this as a transitional pattern on the way to full database-per-service isolation.
