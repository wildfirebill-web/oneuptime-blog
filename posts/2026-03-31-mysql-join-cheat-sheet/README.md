# MySQL JOIN Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Join, Query, Cheat Sheet

Description: A quick reference for all MySQL JOIN types with syntax examples - INNER, LEFT, RIGHT, CROSS, SELF, and multi-table joins explained clearly.

---

## JOIN Types at a Glance

```text
INNER JOIN  - rows matching in both tables
LEFT JOIN   - all rows from left, matching from right (NULL if no match)
RIGHT JOIN  - all rows from right, matching from left (NULL if no match)
CROSS JOIN  - every combination of rows (Cartesian product)
SELF JOIN   - a table joined to itself
```

## INNER JOIN

Returns only rows where the join condition is satisfied in both tables.

```sql
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;
```

## LEFT JOIN (LEFT OUTER JOIN)

Returns all rows from the left table. Columns from the right table are NULL when no match exists.

```sql
-- Find employees without a department
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id
WHERE d.id IS NULL;
```

## RIGHT JOIN (RIGHT OUTER JOIN)

Returns all rows from the right table.

```sql
SELECT e.name, d.dept_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.id;
```

## CROSS JOIN

Produces the Cartesian product - every row from the left paired with every row from the right.

```sql
SELECT a.color, b.size
FROM colors a
CROSS JOIN sizes b;
```

## SELF JOIN

Join a table to itself, typically using different aliases.

```sql
-- Find employees and their managers
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

## Multi-Table JOIN

```sql
SELECT o.id, c.name, p.product_name, oi.quantity
FROM orders o
INNER JOIN customers  c  ON o.customer_id  = c.id
INNER JOIN order_items oi ON oi.order_id   = o.id
INNER JOIN products   p  ON oi.product_id  = p.id
WHERE o.status = 'SHIPPED';
```

## JOIN with Aggregate

```sql
SELECT d.dept_name,
       COUNT(e.id)    AS headcount,
       AVG(e.salary)  AS avg_salary
FROM departments d
LEFT JOIN employees e ON e.dept_id = d.id
GROUP BY d.dept_name
ORDER BY headcount DESC;
```

## Joining on Multiple Columns

```sql
SELECT *
FROM order_items a
INNER JOIN inventory b
  ON a.product_id = b.product_id
 AND a.warehouse_id = b.warehouse_id;
```

## USING Clause (shorthand for equal column names)

```sql
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d USING (dept_id);
```

## NATURAL JOIN (use with caution)

Automatically joins on all columns with the same name. Fragile - avoid in production.

```sql
SELECT * FROM employees NATURAL JOIN departments;
```

## Anti-Join Pattern (rows that have no match)

```sql
-- Orders with no items
SELECT o.id
FROM orders o
LEFT JOIN order_items oi ON oi.order_id = o.id
WHERE oi.id IS NULL;
```

## Summary

MySQL supports INNER, LEFT, RIGHT, CROSS, and SELF joins. INNER JOIN is the most common and returns only matched rows. LEFT JOIN is invaluable for finding orphaned or unmatched records. Use multi-table joins with explicit ON conditions for readability, and consider the USING clause when column names match exactly across tables.
