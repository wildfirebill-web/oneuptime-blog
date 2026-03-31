# How to Apply First Normal Form (1NF) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Normalization, 1NF, Schema Design, Relational

Description: Learn what First Normal Form (1NF) means in MySQL and how to refactor tables that violate it by eliminating repeating groups and multi-valued columns.

---

## What Is First Normal Form (1NF)?

A table is in First Normal Form when:

1. Every column contains only atomic (indivisible) values.
2. Every column holds values of a single type.
3. Every column has a unique name.
4. The order of rows does not carry meaning.
5. There are no repeating groups or arrays.

In practice, the most common 1NF violations in MySQL are storing comma-separated lists in a single column or creating numbered column groups like `phone1`, `phone2`, `phone3`.

## Example of a 1NF Violation

```sql
-- BAD: comma-separated values in one column
CREATE TABLE orders (
    id        INT PRIMARY KEY,
    customer  VARCHAR(100),
    items     VARCHAR(500)  -- 'Widget,Gadget,Doohickey'
);
```

This design makes it impossible to query a specific item without string parsing, prevents indexing, and breaks if an item name contains a comma.

Another common violation - repeating groups:

```sql
-- BAD: repeating groups (phone1, phone2, phone3)
CREATE TABLE contacts (
    id     INT PRIMARY KEY,
    name   VARCHAR(100),
    phone1 VARCHAR(20),
    phone2 VARCHAR(20),
    phone3 VARCHAR(20)
);
```

## Applying 1NF: Move Repeating Values to a Child Table

### Before (violating 1NF)

```sql
CREATE TABLE orders (
    id       INT PRIMARY KEY,
    customer VARCHAR(100),
    items    VARCHAR(500)
);

INSERT INTO orders VALUES (1, 'Alice', 'Widget,Gadget,Doohickey');
```

### After (1NF compliant)

```sql
CREATE TABLE orders (
    id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer VARCHAR(100) NOT NULL
);

CREATE TABLE order_items (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id   INT UNSIGNED NOT NULL,
    item_name  VARCHAR(100) NOT NULL,
    CONSTRAINT fk_items_order FOREIGN KEY (order_id) REFERENCES orders (id) ON DELETE CASCADE
);

INSERT INTO orders (customer) VALUES ('Alice');
INSERT INTO order_items (order_id, item_name) VALUES
    (1, 'Widget'),
    (1, 'Gadget'),
    (1, 'Doohickey');
```

### Fixing the Repeating Phone Columns

```sql
-- 1NF compliant: separate table for phone numbers
CREATE TABLE contacts (
    id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE contact_phones (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    contact_id INT UNSIGNED NOT NULL,
    phone      VARCHAR(20)  NOT NULL,
    label      VARCHAR(20)  NOT NULL DEFAULT 'mobile',  -- 'home', 'work', 'mobile'
    CONSTRAINT fk_phones_contact FOREIGN KEY (contact_id) REFERENCES contacts (id) ON DELETE CASCADE
);
```

## Querying the 1NF Schema

```sql
-- Find all items for order 1
SELECT item_name FROM order_items WHERE order_id = 1;

-- Count items per order
SELECT o.customer, COUNT(oi.id) AS item_count
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.customer;
```

## Benefits of 1NF

- SQL `WHERE` clauses and indexes work correctly on individual values.
- Adding a fourth, fifth, or nth item requires no schema change.
- Aggregations (`COUNT`, `SUM`, `GROUP BY`) work naturally.

## Summary

Achieve 1NF by ensuring every column stores exactly one value. Replace comma-separated lists and numbered column groups with child tables joined by foreign keys. This foundational step makes queries predictable and enables further normalization toward 2NF and 3NF.
