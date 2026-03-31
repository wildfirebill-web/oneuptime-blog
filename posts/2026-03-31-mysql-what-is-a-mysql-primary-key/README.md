# What Is a MySQL Primary Key

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Primary Key, InnoDB

Description: Understand what a MySQL primary key is, how InnoDB stores data around it, how to choose the right primary key type, and best practices for primary key design.

---

A primary key is a column or set of columns that uniquely identifies each row in a MySQL table. Every InnoDB table should have a primary key - it is not just a constraint, it is the physical organization of the table's data on disk.

## How InnoDB Uses the Primary Key

InnoDB is a clustered index storage engine. The primary key defines the physical order in which rows are stored. The table itself is the index - called the clustered index. This means looking up a row by primary key is the fastest possible access.

```sql
CREATE TABLE users (
  id    INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  name  VARCHAR(100) NOT NULL
) ENGINE=InnoDB;

-- Primary key lookup: direct access to row data
SELECT * FROM users WHERE id = 42;
-- EXPLAIN: type=const (single row lookup via primary key)
```

## Defining a Primary Key

```sql
-- Single-column primary key
CREATE TABLE products (
  id   INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL
);

-- Composite primary key (defined separately)
CREATE TABLE order_items (
  order_id   INT NOT NULL,
  product_id INT NOT NULL,
  quantity   INT NOT NULL DEFAULT 1,
  PRIMARY KEY (order_id, product_id)
);
```

## AUTO_INCREMENT Primary Keys

`AUTO_INCREMENT` generates a sequential integer value automatically. MySQL guarantees each inserted row gets a value larger than all previous values (within a session, values do not repeat even after rollbacks).

```sql
INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice');
SELECT LAST_INSERT_ID(); -- Returns the generated ID

-- Reset AUTO_INCREMENT
ALTER TABLE users AUTO_INCREMENT = 1000;
```

## UUID vs AUTO_INCREMENT Primary Keys

UUIDs are globally unique and suitable for distributed systems, but they have trade-offs in InnoDB.

```sql
-- UUID as primary key (common but problematic for InnoDB)
CREATE TABLE events (
  id   CHAR(36) DEFAULT (UUID()) PRIMARY KEY,
  name VARCHAR(255)
);

-- Better: store UUID as binary (MySQL 8.0+)
CREATE TABLE events (
  id   BINARY(16) DEFAULT (UUID_TO_BIN(UUID())) PRIMARY KEY,
  name VARCHAR(255)
);
SELECT BIN_TO_UUID(id), name FROM events;
```

Random UUIDs cause page fragmentation in InnoDB's clustered index because new rows insert at random positions rather than at the end. This leads to frequent B-tree page splits. Use UUIDv7 or `UUID_TO_BIN(uuid, 1)` (swap flag) to preserve time-ordering.

## What Happens Without a Primary Key

If you create an InnoDB table without a primary key, MySQL creates a hidden 6-byte `DB_ROW_ID` column as the clustered index. This is invisible and cannot be referenced in queries.

```sql
-- Avoid this - no primary key
CREATE TABLE bad_design (
  name VARCHAR(100),
  data TEXT
) ENGINE=InnoDB;
-- MySQL silently creates a hidden clustered index
-- You cannot reference or use it in WHERE clauses
```

## Composite Primary Keys

For junction tables representing many-to-many relationships, composite primary keys are appropriate.

```sql
CREATE TABLE user_roles (
  user_id INT NOT NULL,
  role_id INT NOT NULL,
  granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
) ENGINE=InnoDB;
```

## Summary

The MySQL primary key is the clustered index in InnoDB - it physically organizes rows on disk and determines the fastest access path. Always define an explicit primary key. Use `AUTO_INCREMENT INT` or `BIGINT` for most tables. Use ordered UUIDs (UUIDv7 or BIN with swap flag) when global uniqueness across systems is required. Composite primary keys are appropriate for junction tables. A table without a primary key has degraded performance and a hidden row ID that cannot be queried.
