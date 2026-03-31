# How to Calculate Cumulative Sums Over Time in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cumulative Sum, Window Function, runningAccumulate, Analytics

Description: Learn how to calculate cumulative sums and running totals over time in ClickHouse using window functions, runningAccumulate, and arrayCumSum.

---

## Cumulative Sums in Analytics

Cumulative sums show total progress over time: total revenue to date, cumulative users, running error counts. ClickHouse provides multiple approaches depending on the data shape and performance needs.

## Window Function Approach

The cleanest approach uses `sum() OVER` with a growing window:

```sql
SELECT
    toDate(event_time) AS day,
    sum(revenue) AS daily_revenue,
    sum(sum(revenue)) OVER (ORDER BY toDate(event_time)) AS cumulative_revenue
FROM sales
WHERE event_time >= toDate('2026-01-01')
GROUP BY day
ORDER BY day;
```

The nested `sum(sum(...)) OVER` first groups by day, then sums all previous days in the window.

## runningAccumulate

For very large datasets, `runningAccumulate` is more efficient than a window function:

```sql
SELECT
    day,
    daily_revenue,
    runningAccumulate(cumsum_state) AS cumulative_revenue
FROM (
    SELECT
        toDate(event_time) AS day,
        sum(revenue) AS daily_revenue,
        sumState(revenue) AS cumsum_state
    FROM sales
    WHERE event_time >= toDate('2026-01-01')
    GROUP BY day
    ORDER BY day
);
```

## Cumulative Count (Running Signups)

Track cumulative user registrations:

```sql
SELECT
    toDate(created_at) AS day,
    count() AS new_users,
    sum(count()) OVER (ORDER BY toDate(created_at)) AS total_users
FROM users
WHERE created_at >= toDate('2026-01-01')
GROUP BY day
ORDER BY day;
```

## Year-to-Date Cumulative

Calculate YTD metrics resetting each year:

```sql
SELECT
    toDate(event_time) AS day,
    toYear(event_time) AS year,
    sum(revenue) AS daily_revenue,
    sum(sum(revenue)) OVER (
        PARTITION BY toYear(event_time)
        ORDER BY toDate(event_time)
    ) AS ytd_revenue
FROM sales
WHERE event_time >= toDate('2025-01-01')
GROUP BY day, year
ORDER BY day;
```

## arrayCumSum for Pre-Aggregated Arrays

Compute cumulative sums over arrays without row-level window functions:

```sql
SELECT
    service,
    groupArray(daily_errors) AS errors_array,
    arrayCumSum(groupArray(daily_errors)) AS cumulative_errors
FROM (
    SELECT service, toDate(ts) AS day, count() AS daily_errors
    FROM error_logs
    WHERE ts >= today() - 30
    GROUP BY service, day
    ORDER BY service, day
)
GROUP BY service;
```

## Summary

ClickHouse calculates cumulative sums using `sum() OVER` window functions for clarity, `runningAccumulate` with state functions for performance, and `arrayCumSum` for array-based aggregations. Partition-based windows reset the cumulative sum per year or other period for YTD and MTD analytics.
