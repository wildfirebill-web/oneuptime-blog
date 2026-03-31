# How to Use BETWEEN Operator in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, BETWEEN, Range Query, SQL, Index

Description: Learn how to use the BETWEEN operator in ClickHouse for range queries on dates, numbers, and strings, with tips on index utilization.

---

## Basic BETWEEN Syntax

`BETWEEN low AND high` is inclusive on both ends. It is equivalent to `>= low AND <= high`.

```sql
SELECT order_id, amount, order_date
FROM orders
WHERE amount BETWEEN 100 AND 500
  AND order_date BETWEEN '2025-01-01' AND '2025-12-31'
```

## Date Range Queries

Date and DateTime ranges are common in ClickHouse analytical queries. BETWEEN works naturally with date literals.

```sql
SELECT
  toStartOfDay(event_time) AS day,
  count() AS events
FROM events
WHERE event_time BETWEEN '2025-06-01 00:00:00' AND '2025-06-30 23:59:59'
GROUP BY day
ORDER BY day
```

Using the `event_date` partition column (if it exists) is faster than filtering on `event_time` alone because it allows partition pruning.

```sql
-- Better: leverage partition column for pruning
SELECT count()
FROM events
WHERE event_date BETWEEN '2025-06-01' AND '2025-06-30'
```

## NOT BETWEEN

```sql
SELECT user_id, score
FROM user_scores
WHERE score NOT BETWEEN 40 AND 60
-- Equivalent to: score < 40 OR score > 60
```

## BETWEEN with Integer Ranges

```sql
SELECT *
FROM products
WHERE price_cents BETWEEN 1000 AND 5000   -- $10 to $50
  AND stock_count BETWEEN 1 AND 999
```

## Index Utilization with BETWEEN

ClickHouse uses sparse primary key indexes to skip granules. If your table has `ORDER BY (user_id, event_date)`, a range query on `event_date` alone may not benefit from the index unless `user_id` is also filtered. Put the most selective leading key column in your filter first.

```sql
-- Good: filters on leading key columns
SELECT * FROM events
WHERE user_id = 12345
  AND event_date BETWEEN '2025-01-01' AND '2025-03-31'

-- Less efficient: skips leading column
SELECT * FROM events
WHERE event_date BETWEEN '2025-01-01' AND '2025-03-31'
```

## Combining BETWEEN with Other Filters

```sql
SELECT
  user_id,
  event_type,
  revenue
FROM events
WHERE event_date BETWEEN '2025-01-01' AND '2025-06-30'
  AND revenue BETWEEN 0.01 AND 10000
  AND event_type IN ('purchase', 'subscription')
```

## Summary

BETWEEN in ClickHouse is a readable shorthand for inclusive range filters. Use it with date columns alongside partition key columns to enable partition pruning. Place leading sort-key columns in your WHERE clause before range columns to maximize index efficiency. `NOT BETWEEN` is equivalent to two separate inequality conditions.
