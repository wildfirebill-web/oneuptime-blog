# How to Structure MySQL Databases for Maintainability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Schema, Best Practice, Migration, Architecture

Description: Learn how to structure MySQL databases for long-term maintainability by applying normalization, versioned migrations, audit columns, and schema documentation practices.

---

A well-structured MySQL database is easy to understand, safe to modify, and consistent to query. Structural decisions made early - normalization, audit columns, migration tooling - determine how much pain teams experience as the schema evolves.

## Normalize to at Least Third Normal Form

Start with proper normalization to eliminate redundancy:

```sql
-- Bad: stores city and country directly on users
CREATE TABLE users (
  id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name    VARCHAR(200),
  city    VARCHAR(100),
  country VARCHAR(100)
);

-- Better: extracted to a reference table
CREATE TABLE countries (
  id   SMALLINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  code CHAR(2) NOT NULL,
  UNIQUE KEY uq_code (code)
);

CREATE TABLE cities (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name       VARCHAR(100) NOT NULL,
  country_id SMALLINT UNSIGNED NOT NULL,
  CONSTRAINT fk_cities_country FOREIGN KEY (country_id) REFERENCES countries (id)
);

CREATE TABLE users (
  id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name    VARCHAR(200),
  city_id BIGINT UNSIGNED,
  CONSTRAINT fk_users_city FOREIGN KEY (city_id) REFERENCES cities (id)
);
```

## Add Audit Columns to Every Table

Standard timestamps allow you to answer "when was this created or last changed" without additional instrumentation:

```sql
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
created_by BIGINT UNSIGNED,
updated_by BIGINT UNSIGNED
```

For compliance workloads, add soft delete support:

```sql
deleted_at DATETIME DEFAULT NULL,
INDEX idx_deleted_at (deleted_at)
```

## Use Versioned Migrations

Never modify a production schema by hand. Use a migration tool such as Flyway or Liquibase. Each migration is a numbered SQL file:

```text
db/migrations/
  V1__create_users.sql
  V2__create_orders.sql
  V3__add_users_city_id.sql
  V4__create_index_orders_status.sql
```

A sample migration file:

```sql
-- V3__add_users_city_id.sql
ALTER TABLE users
  ADD COLUMN city_id BIGINT UNSIGNED AFTER name,
  ADD CONSTRAINT fk_users_city FOREIGN KEY (city_id) REFERENCES cities (id);
```

Apply migrations with:

```bash
flyway -url=jdbc:mysql://localhost:3306/myapp \
       -user=app_user -password=secret migrate
```

## Group Tables by Domain

In large schemas, prefix tables by bounded context or use separate schemas per domain:

```sql
-- Billing domain
billing_invoices
billing_payments
billing_subscriptions

-- Catalog domain
catalog_products
catalog_categories
catalog_attributes
```

This makes it clear which tables belong together and reduces naming collisions.

## Document Columns with COMMENT

MySQL supports inline comments on columns and tables:

```sql
CREATE TABLE orders (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY COMMENT 'Internal auto-increment PK',
  external_id CHAR(36) NOT NULL COMMENT 'UUID exposed to clients and external systems',
  status      VARCHAR(50) NOT NULL COMMENT 'Values: pending, processing, shipped, cancelled',
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) COMMENT='Customer orders. Append-only; use status updates instead of deletes.';
```

Retrieve comments to generate documentation:

```sql
SELECT COLUMN_NAME, COLUMN_COMMENT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp' AND TABLE_NAME = 'orders';
```

## Summary

Maintainable MySQL schemas share common traits: normalized structure to eliminate redundancy, audit columns on every table, versioned migrations tracked in version control, domain-grouped table names, and inline documentation via COMMENT. These practices make it safe for multiple developers to evolve the schema over years without accumulating technical debt.
