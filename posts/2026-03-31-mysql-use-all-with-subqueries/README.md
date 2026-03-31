# How to Use ALL with Subqueries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ALL, Subquery, Comparison, Query

Description: Learn how to use the ALL operator with subqueries in MySQL to compare a value against every result returned by an inner query.

---

## What Is the ALL Operator

The `ALL` operator compares a value against all values returned by a subquery. The condition is true only if the comparison holds for every value in the subquery result. If the subquery returns no rows, `ALL` evaluates to `TRUE`.

```sql
value comparison_operator ALL (subquery)
```

Supported comparison operators: `=`, `!=`, `<`, `<=`, `>`, `>=`.

## Finding Rows Greater Than All Subquery Values

Find products whose price is higher than every product in the 'Accessories' category:

```sql
SELECT id, name, price
FROM products
WHERE price > ALL (
  SELECT price FROM products WHERE category = 'Accessories'
);
```

This is equivalent to asking: "Is this product's price greater than the maximum price in Accessories?" You could also write:

```sql
SELECT id, name, price
FROM products
WHERE price > (SELECT MAX(price) FROM products WHERE category = 'Accessories');
```

The `ALL` form is more expressive and scales to other comparisons without needing to pick the right aggregate.

## Finding Rows That Do Not Equal Any Subquery Value

`!= ALL` is equivalent to `NOT IN`:

```sql
-- Find orders not in the flagged list
SELECT id, amount
FROM orders
WHERE id != ALL (SELECT order_id FROM flagged_orders);

-- Equivalent using NOT IN
SELECT id, amount
FROM orders
WHERE id NOT IN (SELECT order_id FROM flagged_orders);
```

Unlike `NOT IN`, `!= ALL` is safe when the subquery can return `NULL`: if the subquery returns any `NULL`, `NOT IN` returns no rows, but `!= ALL` handles it consistently.

## Finding Rows Less Than or Equal to All Subquery Values

Find the employee who earns the least across all departments:

```sql
SELECT id, name, salary
FROM employees
WHERE salary <= ALL (SELECT salary FROM employees);
```

This returns the minimum salary row(s).

## ALL with = Operator

`= ALL` is true only when the value equals every result from the subquery, which is only meaningful when the subquery returns a single distinct value:

```sql
-- Check if all orders for a user have the same status
SELECT user_id
FROM orders
GROUP BY user_id
HAVING 'completed' = ALL (SELECT DISTINCT status FROM orders WHERE user_id = orders.user_id);
```

## Performance Considerations

- `ALL` with a large subquery result set can be slow. MySQL must compare against every row.
- Prefer rewriting with `MAX()` or `MIN()` aggregates where possible for better optimizer hints.
- Use `EXPLAIN` to verify the query plan.

```sql
EXPLAIN
SELECT id, name, price
FROM products
WHERE price > ALL (SELECT price FROM products WHERE category = 'Accessories');
```

## Summary

The `ALL` operator in MySQL compares a value against every row returned by a subquery. Use `> ALL` for "greater than the maximum", `< ALL` for "less than the minimum", and `!= ALL` as a NULL-safe alternative to `NOT IN`. For large subqueries, consider rewriting with aggregate functions for better performance.
