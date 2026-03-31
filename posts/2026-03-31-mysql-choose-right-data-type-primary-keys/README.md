# How to Choose the Right Data Type for Primary Keys in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Primary Key, Data Type

Description: Learn how to choose the right data type for primary keys in MySQL, covering INT, BIGINT, UUID, and performance trade-offs for each option.

---

Choosing the right data type for your primary key is one of the most important schema decisions you will make. It affects storage, index performance, replication, and how your application generates IDs. This guide walks through the most common options and when to use each.

## INT and BIGINT Auto-Increment

The classic choice for primary keys is an auto-incrementing integer. It is compact, fast to index, and sequential, which is ideal for InnoDB's clustered index.

```sql
CREATE TABLE orders (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT,
  customer_id INT UNSIGNED NOT NULL,
  total DECIMAL(10, 2),
  PRIMARY KEY (id)
);
```

Use `INT UNSIGNED` for up to ~4.3 billion rows. If your table could exceed that, switch to `BIGINT UNSIGNED`:

```sql
CREATE TABLE events (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  event_time DATETIME NOT NULL,
  payload TEXT,
  PRIMARY KEY (id)
);
```

`BIGINT` uses 8 bytes versus 4 bytes for `INT`, but the sequential nature means inserts cluster well in the B-tree index, minimizing page splits.

## CHAR(36) UUID vs BINARY(16) UUID

UUIDs are globally unique, which simplifies distributed systems and merging data from multiple sources. However, the standard string UUID (CHAR(36)) is poor for performance:

```sql
-- Poor choice: random UUIDs cause index fragmentation
CREATE TABLE sessions (
  id CHAR(36) NOT NULL DEFAULT (UUID()),
  user_id INT UNSIGNED NOT NULL,
  PRIMARY KEY (id)
);
```

Store UUIDs as `BINARY(16)` instead to halve storage and improve index locality:

```sql
CREATE TABLE sessions (
  id BINARY(16) NOT NULL DEFAULT (UUID_TO_BIN(UUID(), 1)),
  user_id INT UNSIGNED NOT NULL,
  PRIMARY KEY (id)
);

-- Insert and retrieve
INSERT INTO sessions (id, user_id) VALUES (UUID_TO_BIN(UUID(), 1), 42);
SELECT BIN_TO_UUID(id, 1) AS id, user_id FROM sessions;
```

The second argument `1` to `UUID_TO_BIN` reorders the time-high field to make UUIDs monotonically increasing, dramatically reducing index fragmentation.

## Composite Primary Keys

Some tables naturally have a composite primary key, such as junction tables for many-to-many relationships:

```sql
CREATE TABLE user_roles (
  user_id INT UNSIGNED NOT NULL,
  role_id SMALLINT UNSIGNED NOT NULL,
  assigned_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, role_id)
);
```

Put the most frequently filtered column first. Queries filtering only on `user_id` will use the index, but queries filtering only on `role_id` will not.

## Choosing Based on Use Case

```text
Use Case                        Recommended Type
------------------------------  ----------------------
Single-server, high write load  INT/BIGINT AUTO_INCREMENT
Distributed or multi-master     BINARY(16) UUID ordered
Small lookup table              TINYINT or SMALLINT
Junction/mapping table          Composite INT columns
External ID (e.g. API key)      VARCHAR or BINARY(n)
```

Avoid using `VARCHAR` as a primary key unless the value is inherently unique and bounded in length, such as an ISO country code. Variable-length keys increase index overhead and can slow joins significantly.

## Summary

For most single-server applications, `INT UNSIGNED AUTO_INCREMENT` or `BIGINT UNSIGNED AUTO_INCREMENT` is the right choice - it is fast, compact, and sequential. For distributed systems where IDs must be globally unique, use `BINARY(16)` with time-ordered UUID v1 via `UUID_TO_BIN(UUID(), 1)`. Avoid plain CHAR(36) UUIDs as primary keys due to fragmentation. Composite keys work well for junction tables where the natural key is already unique.
