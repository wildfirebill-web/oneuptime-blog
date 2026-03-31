# How to Use ON DELETE CASCADE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Foreign Key, DDL, SQL, Referential Integrity

Description: Learn how to configure ON DELETE CASCADE in MySQL foreign keys so that deleting a parent row automatically removes all related child rows.

---

## What Is ON DELETE CASCADE?

When you define a foreign key with `ON DELETE CASCADE`, MySQL automatically deletes all child rows whenever the referenced parent row is deleted. Without this option, attempting to delete a parent row that has child rows results in an error:

```text
ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails
```

`ON DELETE CASCADE` delegates the cleanup to the database engine itself.

## Setting Up an Example

```sql
CREATE TABLE customers (
  customer_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  full_name   VARCHAR(100) NOT NULL,
  email       VARCHAR(255) NOT NULL UNIQUE
) ENGINE=InnoDB;

CREATE TABLE orders (
  order_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id INT UNSIGNED NOT NULL,
  total       DECIMAL(10, 2) NOT NULL,
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers (customer_id)
    ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE order_items (
  item_id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  order_id   INT UNSIGNED NOT NULL,
  product_id INT UNSIGNED NOT NULL,
  quantity   INT NOT NULL,
  CONSTRAINT fk_items_order
    FOREIGN KEY (order_id) REFERENCES orders (order_id)
    ON DELETE CASCADE
) ENGINE=InnoDB;
```

## Cascading Deletes in Action

Insert some data and then delete a customer:

```sql
INSERT INTO customers VALUES (1, 'Alice Chen', 'alice@example.com');

INSERT INTO orders (customer_id, total) VALUES (1, 150.00), (1, 89.50);

INSERT INTO order_items (order_id, product_id, quantity) VALUES
  (1, 10, 2), (1, 20, 1), (2, 30, 3);

-- Deleting the customer cascades to orders, which cascades to order_items
DELETE FROM customers WHERE customer_id = 1;
```

After this single `DELETE`, all rows in `orders` and `order_items` that belong to customer 1 are automatically removed. The cascade travels through the chain: `customers` -> `orders` -> `order_items`.

## Multi-Level Cascades

MySQL follows the full cascade chain. If `order_items` itself had child tables with `ON DELETE CASCADE`, those would be deleted too. There is no enforced depth limit, though deeply nested cascades can be slow on large datasets.

## Verifying the Cascade Worked

```sql
SELECT * FROM customers WHERE customer_id = 1;    -- 0 rows
SELECT * FROM orders WHERE customer_id = 1;        -- 0 rows
SELECT * FROM order_items WHERE order_id IN (1, 2); -- 0 rows
```

## When to Use ON DELETE CASCADE

`ON DELETE CASCADE` is appropriate when child data has no meaning without the parent:

- Order items without an order
- Comments on a deleted post
- Session data for a deleted user
- Audit log entries for a deleted record (debatable)

Avoid it when child data should be preserved independently or when accidental parent deletions would cause irreversible data loss at scale.

## Adding CASCADE to an Existing Table

If the table already has a foreign key without cascade behavior, update it:

```sql
ALTER TABLE orders DROP FOREIGN KEY fk_orders_customer;

ALTER TABLE orders
  ADD CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers (customer_id)
    ON DELETE CASCADE;
```

## ON DELETE CASCADE vs. ON DELETE SET NULL

| Behavior | Result on child row |
|---|---|
| ON DELETE CASCADE | Child row is deleted |
| ON DELETE SET NULL | Child FK column is set to NULL |
| ON DELETE RESTRICT | Parent delete is rejected (default) |
| ON DELETE NO ACTION | Same as RESTRICT in MySQL |

## Performance Note

Cascade deletes acquire locks on all affected child rows. On tables with millions of rows, a single parent delete can lock a large portion of a table. For bulk deletes, consider deleting children explicitly first, or batching deletes.

## Summary

`ON DELETE CASCADE` is a powerful MySQL foreign key option that automates referential cleanup when parent rows are deleted. Define it in the `FOREIGN KEY` constraint declaration and it propagates through as many levels of child tables as needed. Use it for tightly coupled parent-child data and pair it with proper indexing on FK columns to keep cascades efficient.
