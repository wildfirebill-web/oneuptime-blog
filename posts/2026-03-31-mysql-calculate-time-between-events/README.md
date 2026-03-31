# How to Calculate Time Between Events in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, LAG, Window Function, TIMESTAMPDIFF, Analytics

Description: Learn how to calculate elapsed time between consecutive events in MySQL using LAG(), TIMESTAMPDIFF(), and LEAD() window functions for funnel and session analysis.

---

## Why Calculate Time Between Events?

Time-between-events analysis drives funnel optimization, session duration tracking, and SLA compliance reporting. MySQL `LAG()` and `LEAD()` window functions make these calculations efficient without correlated subqueries.

## Basic Time Between Consecutive Events

Use `LAG()` to get the previous event timestamp and `TIMESTAMPDIFF()` to calculate the interval:

```sql
SELECT
  user_id,
  event_type,
  event_time,
  LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event_time,
  TIMESTAMPDIFF(
    SECOND,
    LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time),
    event_time
  ) AS seconds_since_last_event
FROM user_events
ORDER BY user_id, event_time;
```

## Calculating Session Duration

Find the duration of each user session from first to last event:

```sql
SELECT
  user_id,
  session_id,
  MIN(event_time)                             AS session_start,
  MAX(event_time)                             AS session_end,
  TIMESTAMPDIFF(SECOND, MIN(event_time), MAX(event_time)) AS duration_seconds
FROM user_events
GROUP BY user_id, session_id
ORDER BY user_id, session_start;
```

## Funnel Step Timing

Measure how long users take to move between funnel steps:

```sql
WITH funnel AS (
  SELECT
    user_id,
    MAX(CASE WHEN event_type = 'page_view'   THEN event_time END) AS step1_time,
    MAX(CASE WHEN event_type = 'add_to_cart' THEN event_time END) AS step2_time,
    MAX(CASE WHEN event_type = 'checkout'    THEN event_time END) AS step3_time,
    MAX(CASE WHEN event_type = 'purchase'    THEN event_time END) AS step4_time
  FROM user_events
  GROUP BY user_id
)
SELECT
  user_id,
  TIMESTAMPDIFF(MINUTE, step1_time, step2_time) AS view_to_cart_min,
  TIMESTAMPDIFF(MINUTE, step2_time, step3_time) AS cart_to_checkout_min,
  TIMESTAMPDIFF(MINUTE, step3_time, step4_time) AS checkout_to_purchase_min
FROM funnel
WHERE step1_time IS NOT NULL AND step4_time IS NOT NULL
ORDER BY user_id;
```

## Average Time Between Events

Compute the average gap between order placements per customer:

```sql
WITH order_gaps AS (
  SELECT
    customer_id,
    order_date,
    LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_date,
    DATEDIFF(
      order_date,
      LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)
    ) AS days_between
  FROM orders
)
SELECT
  customer_id,
  ROUND(AVG(days_between), 1) AS avg_days_between_orders,
  MIN(days_between)           AS min_days_between_orders,
  MAX(days_between)           AS max_days_between_orders
FROM order_gaps
WHERE days_between IS NOT NULL
GROUP BY customer_id
ORDER BY avg_days_between_orders;
```

## Time to First Event (Time to Value)

Measure the elapsed time from registration to first purchase:

```sql
SELECT
  u.id AS user_id,
  u.created_at AS registered_at,
  MIN(o.order_date) AS first_order_date,
  TIMESTAMPDIFF(HOUR, u.created_at, MIN(o.order_date)) AS hours_to_first_purchase
FROM users u
JOIN orders o ON o.customer_id = u.id
GROUP BY u.id, u.created_at
ORDER BY hours_to_first_purchase;
```

## Using LEAD() for Time to Next Event

`LEAD()` looks ahead instead of behind:

```sql
SELECT
  user_id,
  event_type,
  event_time,
  TIMESTAMPDIFF(
    SECOND,
    event_time,
    LEAD(event_time) OVER (PARTITION BY user_id ORDER BY event_time)
  ) AS seconds_to_next_event
FROM user_events;
```

## Summary

Calculate time between events in MySQL using `LAG()` or `LEAD()` to retrieve adjacent timestamps, combined with `TIMESTAMPDIFF()` or `DATEDIFF()` for the interval. Use `PARTITION BY user_id` to track per-user event chains. Aggregate with `AVG`, `MIN`, and `MAX` to build funnel timing reports and engagement metrics.
