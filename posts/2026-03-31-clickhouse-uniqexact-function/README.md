# How to Use uniqExact() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, uniqExact, Count Distinct

Description: Learn how uniqExact() computes exact distinct counts in ClickHouse, its performance tradeoffs versus uniq(), and when precise cardinality is required.

---

While `uniq()` provides fast approximate distinct counts, some use cases require a guaranteed exact result. ClickHouse's `uniqExact()` function counts the precise number of distinct values by maintaining a hash set of every unique value seen. This delivers 100% accurate results at the cost of higher memory usage and slower execution compared to HyperLogLog-based alternatives.

## Basic Syntax

```sql
-- Exact count of distinct users
SELECT uniqExact(user_id) AS exact_distinct_users
FROM page_views;
```

You can pass multiple columns to count distinct combinations exactly:

```sql
-- Exact count of distinct (user_id, device_id) pairs
SELECT uniqExact(user_id, device_id) AS exact_user_device_pairs
FROM sessions;
```

## How uniqExact() Works

Internally, `uniqExact()` stores every distinct value in a hash set. When the query completes, the size of the hash set is returned as the result.

```sql
-- Memory grows with cardinality
-- Safe for columns with bounded or small distinct counts
SELECT uniqExact(country_code) AS exact_countries
FROM user_events;

-- Potentially expensive on high-cardinality columns with many rows
SELECT uniqExact(raw_url) AS exact_urls
FROM access_logs
WHERE event_date >= today() - 30;
```

For very high-cardinality columns (UUIDs, IP addresses, full URLs) over large time ranges, `uniqExact()` can consume significant memory. Consider using `uniq()` or `uniqCombined()` in those situations.

## Comparing uniqExact() and uniq()

```sql
-- Side-by-side comparison
SELECT
    uniq(user_id)      AS approx_users,
    uniqExact(user_id) AS exact_users
FROM user_events
WHERE event_date = today();
```

For datasets with millions of rows, `uniq()` is typically many times faster. The approximate error is usually under 2.2%, which is acceptable for most analytics dashboards. Use `uniqExact()` when that margin is not acceptable.

## When to Use uniqExact()

Choose `uniqExact()` over `uniq()` when:

- You are reporting billable metrics (unique paid users, licensed seats)
- Audit reports require exact figures
- The distinct count is the primary business KPI
- The dataset is small enough that exact counting is fast

```sql
-- Billing report: exact paying customers this month
SELECT
    plan_tier,
    uniqExact(customer_id) AS exact_paying_customers
FROM subscription_events
WHERE event_date >= toStartOfMonth(today())
  AND event_type = 'charge_succeeded'
GROUP BY plan_tier;
```

## Grouping Example

```sql
-- Exact daily active users per platform
SELECT
    toDate(event_time) AS date,
    platform,
    uniqExact(user_id) AS dau
FROM user_events
WHERE event_date >= today() - 7
GROUP BY date, platform
ORDER BY date DESC, dau DESC;
```

## Exact Distinct on Multiple Columns

```sql
-- Count exactly how many unique (user, country) combinations were seen
SELECT uniqExact(user_id, country) AS exact_user_country_pairs
FROM login_events
WHERE event_date = today();
```

This is useful for detecting account sharing across regions or verifying multi-dimensional uniqueness.

## Performance Benchmark Pattern

```sql
-- Measure difference in query time for your data
SELECT
    count()            AS total_rows,
    uniq(session_id)   AS approx_sessions,
    uniqExact(session_id) AS exact_sessions
FROM web_sessions
WHERE event_date >= today() - 1;
```

Run this with `clickhouse-client --time` to measure the wall-clock difference on your dataset.

## Summary

`uniqExact()` delivers precise distinct counts by maintaining a full hash set of every observed value. It is the right choice for billing, compliance, and auditing scenarios where approximate results are not acceptable. For large-scale analytics and high-cardinality columns over long time ranges, prefer `uniq()` or `uniqCombined()` to keep memory usage and query latency under control.
