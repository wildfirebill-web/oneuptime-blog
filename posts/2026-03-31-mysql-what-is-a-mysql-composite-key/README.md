# What Is a MySQL Composite Key

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Composite Key, Index

Description: Learn what a MySQL composite key is, how multi-column primary and unique keys work, the leftmost prefix rule, and when to use composite keys in your schema.

---

A composite key in MySQL is a key (primary key, unique key, or index) that spans two or more columns. It enforces uniqueness or enables efficient lookups based on a combination of column values rather than a single column.

## Composite Primary Key

The most common use of a composite primary key is a junction table in a many-to-many relationship.

```sql
CREATE TABLE user_roles (
  user_id INT NOT NULL,
  role_id INT NOT NULL,
  granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- Each (user_id, role_id) pair must be unique
INSERT INTO user_roles (user_id, role_id) VALUES (1, 3);
INSERT INTO user_roles (user_id, role_id) VALUES (1, 5); -- OK, different role
INSERT INTO user_roles (user_id, role_id) VALUES (1, 3); -- ERROR: duplicate
```

## Composite Unique Key

A composite unique key enforces that the combination of values is unique, not each column individually.

```sql
CREATE TABLE product_attributes (
  product_id  INT NOT NULL,
  attr_name   VARCHAR(100) NOT NULL,
  attr_value  VARCHAR(255),
  UNIQUE KEY uk_product_attr (product_id, attr_name)
) ENGINE=InnoDB;

-- Same product, different attributes are allowed
INSERT INTO product_attributes VALUES (1, 'color', 'red');
INSERT INTO product_attributes VALUES (1, 'size', 'large');

-- Same product, same attribute name is rejected
INSERT INTO product_attributes VALUES (1, 'color', 'blue');
-- ERROR: Duplicate entry '1-color'
```

## Composite Index

Composite indexes are used for query optimization on multi-column WHERE, JOIN, and ORDER BY clauses.

```sql
-- Composite index for queries that filter on both columns
CREATE INDEX idx_status_date ON orders (status, created_at);

-- Efficiently uses the composite index
SELECT * FROM orders
WHERE status = 'pending' AND created_at > '2026-01-01';
```

## The Leftmost Prefix Rule

MySQL can use a composite index only when the query's conditions include the leftmost columns of the index, in order. This is the most important rule for composite index design.

```sql
CREATE INDEX idx_a_b_c ON orders (customer_id, status, created_at);

-- Uses index (leftmost column present)
SELECT * FROM orders WHERE customer_id = 42;

-- Uses index (first two columns)
SELECT * FROM orders WHERE customer_id = 42 AND status = 'paid';

-- Uses index (all three columns)
SELECT * FROM orders WHERE customer_id = 42 AND status = 'paid' AND created_at > '2026-01-01';

-- Does NOT use index (leading column missing)
SELECT * FROM orders WHERE status = 'paid';

-- Does NOT use index efficiently (gap in prefix)
SELECT * FROM orders WHERE customer_id = 42 AND created_at > '2026-01-01';
-- MySQL uses idx for customer_id lookup but not created_at
```

## Column Order in Composite Keys

For composite indexes, put the most selective column first - or the column most commonly used as the sole filter in equality conditions.

```sql
-- If queries commonly filter by customer_id alone, put it first
CREATE INDEX idx_cust_status ON orders (customer_id, status);

-- If most queries filter by both, ordering is less critical
-- but equality columns should precede range columns
CREATE INDEX idx_status_date ON orders (status, created_at);
-- WHERE status = 'paid' AND created_at > ... -- good fit
```

## Viewing Composite Keys

```sql
-- Show all indexes including composite keys
SHOW INDEX FROM orders;

-- Information schema: group columns by index name
SELECT INDEX_NAME, SEQ_IN_INDEX, COLUMN_NAME, NON_UNIQUE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'orders'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
```

## Summary

A composite key spans multiple columns to enforce uniqueness or enable efficient multi-column lookups. Composite primary keys are standard for junction tables. Composite unique keys enforce uniqueness on combinations of values. Composite indexes follow the leftmost prefix rule - queries must include the leading columns to benefit. Place equality filter columns before range filter columns in composite index definitions.
