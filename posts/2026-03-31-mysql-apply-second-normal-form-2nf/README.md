# How to Apply Second Normal Form (2NF) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Normalization, 2NF, Schema Design, Dependency

Description: Learn what Second Normal Form (2NF) means in MySQL and how to eliminate partial dependencies from tables that use composite primary keys.

---

## Prerequisites

A table must already satisfy First Normal Form (1NF) before applying 2NF. 2NF only applies to tables with composite primary keys (primary keys made up of more than one column).

## What Is Second Normal Form (2NF)?

A table is in 2NF when:

1. It is already in 1NF.
2. Every non-key column is fully functionally dependent on the **entire** composite primary key - not just part of it.

A **partial dependency** exists when a non-key column depends on only one column of the composite key.

## Example of a 2NF Violation

Consider an `order_items` table where the primary key is `(order_id, product_id)`:

```sql
-- VIOLATES 2NF: product_name and category depend only on product_id, not the full PK
CREATE TABLE order_items (
    order_id     INT UNSIGNED NOT NULL,
    product_id   INT UNSIGNED NOT NULL,
    qty          INT UNSIGNED NOT NULL,
    unit_price   DECIMAL(10,2) NOT NULL,
    product_name VARCHAR(255),   -- depends only on product_id
    category     VARCHAR(100),   -- depends only on product_id
    PRIMARY KEY (order_id, product_id)
);
```

`product_name` and `category` depend on `product_id` alone. If the product name changes, you must update every row in `order_items` that references that product.

## Applying 2NF: Extract the Partial Dependency

Move `product_name` and `category` into their own `products` table:

```sql
-- Products table: non-key attributes fully depend on product_id
CREATE TABLE products (
    id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name     VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL
);

-- order_items: only columns that require both order_id AND product_id
CREATE TABLE order_items (
    order_id   INT UNSIGNED NOT NULL,
    product_id INT UNSIGNED NOT NULL,
    qty        INT UNSIGNED NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_items_order   FOREIGN KEY (order_id)   REFERENCES orders   (id) ON DELETE CASCADE,
    CONSTRAINT fk_items_product FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE RESTRICT
);
```

Now `unit_price` correctly lives in `order_items` because the price at the time of purchase depends on both the order and the product (a historical snapshot, not the current price).

## Querying the 2NF Schema

```sql
SELECT
    oi.order_id,
    p.name          AS product_name,
    p.category,
    oi.qty,
    oi.unit_price,
    oi.qty * oi.unit_price AS line_total
FROM order_items oi
JOIN products p ON p.id = oi.product_id
WHERE oi.order_id = 1;
```

## Another Example: Student Course Grades

```sql
-- VIOLATES 2NF: teacher_name depends only on course_id
-- BAD:
-- (student_id, course_id) PK | grade | teacher_name

-- 2NF compliant:
CREATE TABLE courses (
    id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    teacher_name VARCHAR(100) NOT NULL
);

CREATE TABLE enrollments (
    student_id INT UNSIGNED NOT NULL,
    course_id  INT UNSIGNED NOT NULL,
    grade      CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

## Summary

2NF eliminates partial dependencies by ensuring non-key columns depend on the full composite key. Identify columns that describe only part of the key and move them to a separate table referenced by the partial key. Tables with a single-column primary key are automatically in 2NF, so 2NF work is only needed when composite keys are present.
