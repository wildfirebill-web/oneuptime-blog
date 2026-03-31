# How to Update Rows Based on Another Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Update, Join, Subquery, DML

Description: Learn how to update rows in a MySQL table using values or conditions from another table with JOIN-based UPDATE and correlated subqueries.

---

## Why Update from Another Table

Relational databases often store related data across multiple tables. When a source table changes, you need to propagate those changes to a dependent table. MySQL supports this through multi-table `UPDATE` syntax (JOIN-based) and correlated subqueries.

## JOIN-Based UPDATE Syntax

The most readable and performant approach is to JOIN the target table with the source table directly in the `UPDATE` statement:

```sql
UPDATE orders o
JOIN customers c ON o.customer_id = c.id
SET o.discount = c.loyalty_discount
WHERE c.tier = 'gold';
```

MySQL updates the `orders` table, reading the `discount` value from the joined `customers` row. Only orders belonging to gold-tier customers are affected.

## Updating from an Aggregated Source

Combine a subquery with a JOIN to update from aggregated data:

```sql
UPDATE products p
JOIN (
  SELECT product_id, SUM(quantity) AS total_sold
  FROM order_items
  GROUP BY product_id
) AS sales ON p.id = sales.product_id
SET p.units_sold = sales.total_sold;
```

The derived table computes the total sold for each product, and the `UPDATE` writes the result back to the `products` table.

## Correlated Subquery in SET

An alternative is a correlated subquery in the `SET` clause:

```sql
UPDATE employees e
SET e.department_name = (
  SELECT d.name
  FROM departments d
  WHERE d.id = e.department_id
);
```

This is functionally equivalent to a JOIN but can be slower for large tables because the subquery executes once per row. Prefer the JOIN form for performance.

## Correlated Subquery in WHERE

Use a subquery in the `WHERE` clause to filter which rows to update based on another table:

```sql
UPDATE inventory
SET reorder_flag = 1
WHERE product_id IN (
  SELECT id FROM products WHERE category = 'perishable'
);
```

## Updating Multiple Columns from Another Table

Set several columns at once from a joined table:

```sql
UPDATE shipments s
JOIN warehouses w ON s.warehouse_id = w.id
SET s.warehouse_name = w.name,
    s.warehouse_city = w.city,
    s.updated_at     = NOW()
WHERE s.status = 'pending';
```

## Preventing Accidental Full-Table Updates

Always verify the JOIN condition returns the expected rows before running the update. Use a matching `SELECT` to preview:

```sql
SELECT o.id, o.discount, c.loyalty_discount
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.tier = 'gold';
```

Once the result looks correct, replace `SELECT ...` with `UPDATE ... SET`.

## Summary

MySQL supports updating rows based on another table using either JOIN syntax in the `UPDATE` statement or correlated subqueries. The JOIN-based multi-table `UPDATE` is typically faster and more readable. Always preview with a matching `SELECT` before executing, and verify the affected row count afterward with `ROW_COUNT()`.
