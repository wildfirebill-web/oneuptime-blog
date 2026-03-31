# How to Translate Oracle SQL to ClickHouse SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Oracle, SQL Translation, Migration, Analytics

Description: Convert Oracle SQL patterns to ClickHouse SQL covering analytic functions, hierarchical queries, DATE types, and Oracle-specific syntax with practical examples.

---

Migrating analytics from Oracle to ClickHouse requires translating Oracle-specific functions, date handling, and analytical SQL patterns. ClickHouse is more expressive for analytics but lacks Oracle's procedural features.

## Date and Time Functions

Oracle's date arithmetic uses different syntax:

```sql
-- Oracle
SELECT SYSDATE, SYSDATE - 7, TO_DATE('2025-01-01', 'YYYY-MM-DD') FROM dual;

-- ClickHouse
SELECT now(), now() - INTERVAL 7 DAY, toDate('2025-01-01');
```

```sql
-- Oracle: truncate to month
SELECT TRUNC(order_date, 'MM') FROM orders;

-- ClickHouse
SELECT toStartOfMonth(order_date) FROM orders;
```

## ROWNUM and ROWID Equivalents

```sql
-- Oracle: top 10 rows
SELECT * FROM events WHERE ROWNUM <= 10;

-- ClickHouse
SELECT * FROM events LIMIT 10;

-- Oracle: row number in window
SELECT ROWNUM, order_id FROM orders;

-- ClickHouse
SELECT row_number() OVER () AS rn, order_id FROM orders;
```

## Analytic Functions

Oracle's `OVER` clause translates directly, but function names differ:

```sql
-- Oracle: running total
SELECT order_id, amount,
    SUM(amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM orders;

-- ClickHouse (identical syntax)
SELECT order_id, amount,
    sum(amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM orders;
```

```sql
-- Oracle LEAD/LAG
SELECT LAG(amount, 1, 0) OVER (ORDER BY ts) AS prev_amount FROM sales;

-- ClickHouse
SELECT lagInFrame(amount, 1, 0) OVER (ORDER BY ts) AS prev_amount FROM sales;
```

## NVL and DECODE

```sql
-- Oracle NVL
SELECT NVL(revenue, 0) FROM sales;

-- ClickHouse
SELECT ifNull(revenue, 0) FROM sales;

-- Oracle DECODE
SELECT DECODE(status, 'A', 'Active', 'I', 'Inactive', 'Unknown') FROM users;

-- ClickHouse
SELECT CASE status WHEN 'A' THEN 'Active' WHEN 'I' THEN 'Inactive' ELSE 'Unknown' END FROM users;
```

## Hierarchical Queries (CONNECT BY)

Oracle's `CONNECT BY` has no direct equivalent. Use recursive CTEs:

```sql
-- Oracle
SELECT employee_id, manager_id, LEVEL
FROM employees
CONNECT BY PRIOR employee_id = manager_id
START WITH manager_id IS NULL;

-- ClickHouse (recursive CTE, available in newer versions)
WITH RECURSIVE emp_tree AS (
    SELECT employee_id, manager_id, 1 AS lvl FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.manager_id, et.lvl + 1
    FROM employees e JOIN emp_tree et ON e.manager_id = et.employee_id
)
SELECT employee_id, manager_id, lvl FROM emp_tree;
```

## FROM dual

```sql
-- Oracle
SELECT 1 + 1 FROM dual;

-- ClickHouse (no dual table needed)
SELECT 1 + 1;
```

## Summary

Oracle-to-ClickHouse translation is mostly mechanical: replace date functions, use `ifNull` for NVL, `lagInFrame` for LAG, and recursive CTEs for hierarchical queries. ClickHouse's columnar engine makes the migrated analytics queries significantly faster on large datasets.
