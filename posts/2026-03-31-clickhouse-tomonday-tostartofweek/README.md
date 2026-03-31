# How to Use toMonday() and toStartOfWeek() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Analytics, Aggregation, Weekly

Description: Learn how to use toMonday() and toStartOfWeek() in ClickHouse to align timestamps to week boundaries for weekly aggregations and comparisons.

---

Weekly aggregations are a staple of business analytics - weekly active users, week-over-week revenue, and weekly error rates all require snapping timestamps to a consistent week boundary. ClickHouse provides two functions for this: `toMonday()` which always returns the Monday of a date's week, and `toStartOfWeek()` which lets you configure the first day of the week.

This post explains both functions, their differences, and shows how to use them for weekly reporting, week-over-week comparison, and grouping.

## toMonday() - Snap to the Monday of the Week

`toMonday(dt)` returns a `Date` value representing the Monday of the ISO week that contains the given date. If the input is already a Monday, it returns that same date.

```sql
SELECT
    toDate('2026-03-31') AS tuesday,
    toMonday(toDate('2026-03-31')) AS week_start;
```

```text
tuesday      week_start
------------ ------------
2026-03-31   2026-03-30
```

March 31, 2026 is a Tuesday, so `toMonday()` returns March 30, 2026.

For a Monday input, the function is a no-op:

```sql
SELECT
    toDate('2026-03-30') AS monday_input,
    toMonday(toDate('2026-03-30')) AS result;
```

```text
monday_input  result
------------- -----------
2026-03-30    2026-03-30
```

## Weekly Aggregations with toMonday()

The primary use case is grouping events by week. Using `toMonday()` in a `GROUP BY` ensures every day of the same ISO week maps to the same bucket:

```sql
SELECT
    toMonday(sale_date)  AS week_start,
    count()              AS orders,
    sum(revenue)         AS weekly_revenue,
    uniq(customer_id)    AS unique_customers
FROM sales
WHERE sale_date >= today() - 90
GROUP BY week_start
ORDER BY week_start;
```

## Week-over-Week Comparison

To compute week-over-week growth, use a window function over the weekly aggregation:

```sql
SELECT
    week_start,
    weekly_revenue,
    lagInFrame(weekly_revenue) OVER (ORDER BY week_start) AS prev_week_revenue,
    round(
        (weekly_revenue - lagInFrame(weekly_revenue) OVER (ORDER BY week_start))
        / lagInFrame(weekly_revenue) OVER (ORDER BY week_start) * 100,
        2
    ) AS wow_growth_pct
FROM (
    SELECT
        toMonday(sale_date) AS week_start,
        sum(revenue)        AS weekly_revenue
    FROM sales
    WHERE sale_date >= today() - 180
    GROUP BY week_start
)
ORDER BY week_start;
```

## toStartOfWeek() - Configurable Week Start Day

`toStartOfWeek(dt, mode)` is more flexible than `toMonday()` because the `mode` argument controls which day is treated as the first day of the week.

```text
Mode 0 (default): Week starts on Sunday
Mode 1:           Week starts on Monday
```

For businesses that use Sunday as the start of the week (common in the United States):

```sql
SELECT
    toDate('2026-03-31') AS tuesday,
    toStartOfWeek(toDate('2026-03-31'), 0) AS sunday_start,
    toStartOfWeek(toDate('2026-03-31'), 1) AS monday_start;
```

```text
tuesday      sunday_start  monday_start
------------ ------------- ------------
2026-03-31   2026-03-29    2026-03-30
```

## Using toStartOfWeek() for Sunday-Based Weekly Reports

If your business week runs Sunday to Saturday, use mode 0:

```sql
SELECT
    toStartOfWeek(event_date, 0)  AS week_start_sunday,
    count()                        AS events,
    sum(revenue)                   AS revenue
FROM sales
WHERE event_date >= today() - 90
GROUP BY week_start_sunday
ORDER BY week_start_sunday;
```

## Comparing toMonday() and toStartOfWeek(mode=1)

`toMonday(dt)` and `toStartOfWeek(dt, 1)` produce identical results. `toMonday()` is shorter and more readable when ISO weeks (Monday start) are always what you need:

```sql
SELECT
    toMonday(toDate('2026-03-31'))          AS via_toMonday,
    toStartOfWeek(toDate('2026-03-31'), 1)  AS via_toStartOfWeek;
```

```text
via_toMonday  via_toStartOfWeek
------------- ------------------
2026-03-30    2026-03-30
```

## Joining This Week to Last Week

A common dashboard pattern is to join this week's data to last week's for a side-by-side table:

```sql
SELECT
    this_week.product_id,
    this_week.revenue        AS this_week_revenue,
    last_week.revenue        AS last_week_revenue,
    this_week.revenue - last_week.revenue AS delta
FROM (
    SELECT product_id, sum(revenue) AS revenue
    FROM sales
    WHERE toMonday(sale_date) = toMonday(today())
    GROUP BY product_id
) AS this_week
LEFT JOIN (
    SELECT product_id, sum(revenue) AS revenue
    FROM sales
    WHERE toMonday(sale_date) = toMonday(today()) - INTERVAL 7 DAY
    GROUP BY product_id
) AS last_week USING (product_id)
ORDER BY delta DESC;
```

## Weekly Active User Count

Count distinct users per week using `toMonday()` as the bucketing key:

```sql
SELECT
    toMonday(event_date)  AS week_start,
    uniq(user_id)         AS weekly_active_users
FROM user_events
WHERE event_date >= today() - 180
GROUP BY week_start
ORDER BY week_start;
```

## Summary

`toMonday()` is the simplest and most readable way to bucket dates into ISO weeks in ClickHouse. Use it whenever your organization defines weeks as Monday through Sunday. When you need to configure a different start day - most commonly Sunday - reach for `toStartOfWeek(dt, mode)` with mode 0. Both functions return a `Date` value that groups cleanly in `GROUP BY`, making weekly aggregations, week-over-week comparisons, and weekly cohort analysis straightforward to express.
