# How to Find Consecutive Sequences in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, ROW_NUMBER, Sequence, Analytics

Description: Learn how to find consecutive sequences of numbers or dates in MySQL using the ROW_NUMBER minus value technique to identify streaks, runs, and contiguous ranges.

---

## What Is a Consecutive Sequence?

A consecutive sequence is a run of values with no gaps - consecutive integers, daily logins without a miss, or sequential IDs. The classic SQL technique is the "ROW_NUMBER minus value" trick, which assigns the same group number to all rows in a run.

## The ROW_NUMBER Minus Value Technique

When you subtract the row number from the value of consecutive integers, the result is constant for all rows in the same run:

```sql
SELECT
  id,
  value,
  ROW_NUMBER() OVER (ORDER BY value) AS rn,
  value - ROW_NUMBER() OVER (ORDER BY value) AS group_key
FROM numbers_table
ORDER BY value;
```

Rows with the same `group_key` belong to the same consecutive sequence.

## Finding All Consecutive Runs

Aggregate by `group_key` to get the start, end, and length of each run:

```sql
WITH ranked AS (
  SELECT
    value,
    value - ROW_NUMBER() OVER (ORDER BY value) AS grp
  FROM numbers_table
)
SELECT
  MIN(value) AS sequence_start,
  MAX(value) AS sequence_end,
  COUNT(*)   AS sequence_length
FROM ranked
GROUP BY grp
ORDER BY sequence_start;
```

## Consecutive Daily Logins (Streak Detection)

Find user login streaks - consecutive days of activity:

```sql
WITH daily_logins AS (
  SELECT DISTINCT user_id, DATE(login_at) AS login_date
  FROM user_sessions
),
ranked AS (
  SELECT
    user_id,
    login_date,
    login_date - INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY AS grp
  FROM daily_logins
)
SELECT
  user_id,
  MIN(login_date) AS streak_start,
  MAX(login_date) AS streak_end,
  COUNT(*)        AS streak_days
FROM ranked
GROUP BY user_id, grp
ORDER BY user_id, streak_start;
```

## Finding the Longest Streak Per User

```sql
WITH daily_logins AS (
  SELECT DISTINCT user_id, DATE(login_at) AS login_date
  FROM user_sessions
),
ranked AS (
  SELECT
    user_id,
    login_date,
    login_date - INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY AS grp
  FROM daily_logins
),
streaks AS (
  SELECT user_id, MIN(login_date) AS start, MAX(login_date) AS end, COUNT(*) AS len
  FROM ranked
  GROUP BY user_id, grp
)
SELECT user_id, MAX(len) AS longest_streak_days
FROM streaks
GROUP BY user_id
ORDER BY longest_streak_days DESC;
```

## Consecutive Sequence of Orders

Find customers with consecutive monthly orders:

```sql
WITH monthly_orders AS (
  SELECT
    customer_id,
    DATE_FORMAT(order_date, '%Y-%m-01') AS order_month
  FROM orders
  GROUP BY customer_id, order_month
),
ranked AS (
  SELECT
    customer_id,
    order_month,
    order_month - INTERVAL ROW_NUMBER() OVER (
      PARTITION BY customer_id ORDER BY order_month
    ) MONTH AS grp
  FROM monthly_orders
)
SELECT
  customer_id,
  MIN(order_month) AS first_month,
  MAX(order_month) AS last_month,
  COUNT(*)         AS consecutive_months
FROM ranked
GROUP BY customer_id, grp
HAVING COUNT(*) >= 3
ORDER BY consecutive_months DESC;
```

## Summary

Find consecutive sequences in MySQL using the "ROW_NUMBER minus value" technique - rows with the same difference value belong to the same contiguous run. For dates, subtract `INTERVAL ROW_NUMBER() DAY` from the date value. Aggregate with `GROUP BY grp` to get the start, end, and length of each run, and use `HAVING` to filter for minimum streak lengths.
