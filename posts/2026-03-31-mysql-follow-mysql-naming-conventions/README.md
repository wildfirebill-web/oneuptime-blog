# How to Follow MySQL Naming Conventions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Schema, Convention, Best Practice, Database

Description: Learn the standard MySQL naming conventions for tables, columns, indexes, and constraints to keep your schema readable, consistent, and easy to maintain.

---

Consistent naming conventions reduce cognitive overhead, make queries more readable, and prevent conflicts caused by reserved words. MySQL is case-insensitive on most platforms but case-sensitive on Linux by default, making a defined convention especially important in cross-platform teams.

## Table Names

Use lowercase, snake_case, plural nouns. Table names represent collections of entities:

```sql
-- Good
CREATE TABLE users (...);
CREATE TABLE order_items (...);
CREATE TABLE product_categories (...);

-- Avoid: mixed case, singular, or abbreviations
CREATE TABLE User (...);
CREATE TABLE ord_itm (...);
```

Avoid reserved words such as `order`, `group`, `key`, and `values` as table names. If you must use them, wrap in backticks - but it is better to rename:

```sql
-- Risky
CREATE TABLE `order` (...);

-- Better
CREATE TABLE orders (...);
```

## Column Names

Use lowercase snake_case for all columns:

```sql
CREATE TABLE orders (
  id             BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id    BIGINT UNSIGNED NOT NULL,
  total_amount   DECIMAL(10,2)  NOT NULL,
  status         VARCHAR(50)    NOT NULL,
  created_at     DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at     DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

Name boolean columns with an `is_` or `has_` prefix:

```sql
is_active       TINYINT(1) NOT NULL DEFAULT 1,
has_subscription TINYINT(1) NOT NULL DEFAULT 0
```

Foreign key columns should mirror the referenced table with a `_id` suffix:

```sql
customer_id BIGINT UNSIGNED NOT NULL  -- references customers.id
```

## Primary Keys

Use `id` for the primary key on every table. Always use `BIGINT UNSIGNED AUTO_INCREMENT`:

```sql
id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
```

For tables that need a business key exposed externally, add a separate `uuid` column alongside the integer `id`:

```sql
id   BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
uuid CHAR(36) NOT NULL DEFAULT (UUID()),
UNIQUE KEY uq_uuid (uuid)
```

## Index Names

Prefix indexes with their type for clarity:

```sql
-- Regular index
ADD INDEX idx_orders_status (status);

-- Unique index
ADD UNIQUE INDEX uq_users_email (email);

-- Foreign key index
ADD INDEX fk_orders_customer_id (customer_id);

-- Full-text index
ADD FULLTEXT INDEX ft_products_name_desc (name, description);
```

## Constraint Names

Name foreign key constraints explicitly so error messages are meaningful:

```sql
ALTER TABLE orders
  ADD CONSTRAINT fk_orders_customer_id
  FOREIGN KEY (customer_id) REFERENCES customers (id)
  ON DELETE RESTRICT ON UPDATE CASCADE;
```

## Stored Objects

Stored procedures use verb-noun form in snake_case. Functions are prefixed with `fn_`:

```sql
CREATE PROCEDURE get_order_by_id(IN p_order_id BIGINT) ...
CREATE FUNCTION fn_calculate_tax(amount DECIMAL) RETURNS DECIMAL ...
```

## Summary

MySQL naming conventions center on lowercase snake_case throughout: plural table names, descriptive column names, prefixed indexes, and explicitly named constraints. Enforcing these conventions through code review or a linting tool keeps schemas readable as teams grow and makes it far easier to trace relationships across tables by name alone.
