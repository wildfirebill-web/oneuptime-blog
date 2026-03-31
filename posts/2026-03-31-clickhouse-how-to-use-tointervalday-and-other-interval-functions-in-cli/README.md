# How to Use toIntervalDay() and Other Interval Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Interval Functions, Date Arithmetic, Sql, Data Engineering

Description: Learn how to use toIntervalDay(), toIntervalMonth(), toIntervalHour() and other interval functions in ClickHouse for date arithmetic.

---

## Overview

ClickHouse provides a family of `toInterval*()` functions that create Interval typed values for performing date and time arithmetic. These are essential for shifting timestamps by a fixed amount - such as adding 7 days, subtracting 3 months, or moving forward 2 hours.

## Available Interval Functions

ClickHouse supports the following interval constructors:

```sql
SELECT
    toIntervalSecond(30)   AS thirty_seconds,
    toIntervalMinute(15)   AS fifteen_minutes,
    toIntervalHour(2)      AS two_hours,
    toIntervalDay(7)       AS one_week,
    toIntervalWeek(1)      AS one_week_alt,
    toIntervalMonth(3)     AS three_months,
    toIntervalQuarter(1)   AS one_quarter,
    toIntervalYear(1)      AS one_year
```

## Adding Intervals to DateTime

Use the `+` operator with an interval to add time to a DateTime:

```sql
SELECT
    now() AS current_time,
    now() + toIntervalDay(7)    AS next_week,
    now() + toIntervalMonth(1)  AS next_month,
    now() - toIntervalHour(24)  AS yesterday
```

You can also add intervals to Date values:

```sql
SELECT
    today() AS today,
    today() + toIntervalDay(30)   AS thirty_days_later,
    today() - toIntervalMonth(6)  AS six_months_ago
```

## Practical Example - Retention Windows

Interval functions are commonly used to define lookback or lookahead windows in analytics queries:

```sql
SELECT
    user_id,
    first_seen,
    first_seen + toIntervalDay(7)   AS day7_window,
    first_seen + toIntervalDay(30)  AS day30_window
FROM
(
    SELECT user_id, min(event_time) AS first_seen
    FROM events
    GROUP BY user_id
)
```

## Filtering with Intervals

Use intervals in WHERE clauses to filter events in a rolling window:

```sql
SELECT count() AS events_last_7_days
FROM events
WHERE event_time >= now() - toIntervalDay(7)
```

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    count() AS events
FROM events
WHERE event_time >= now() - toIntervalHour(48)
GROUP BY hour
ORDER BY hour
```

## Interval Arithmetic in Scheduled Reporting

Interval functions are useful when generating period-over-period comparisons:

```sql
SELECT
    toStartOfDay(now())                         AS today_start,
    toStartOfDay(now()) - toIntervalDay(7)      AS last_week_same_day,
    toStartOfDay(now()) - toIntervalMonth(1)    AS last_month_same_day
```

## Using INTERVAL Keyword Syntax

ClickHouse also supports the SQL standard `INTERVAL` keyword syntax as an alternative:

```sql
SELECT now() + INTERVAL 7 DAY AS next_week;
SELECT now() - INTERVAL 1 MONTH AS last_month;
SELECT now() + INTERVAL 2 HOUR AS two_hours_later;
```

Both syntaxes produce identical results. The `toInterval*()` functions are more explicit and useful when the interval value is a variable.

## Dynamic Intervals from Column Values

You can compute intervals dynamically from table columns:

```sql
SELECT
    subscription_start,
    subscription_length_days,
    subscription_start + toIntervalDay(subscription_length_days) AS subscription_end
FROM subscriptions
```

## Combining Multiple Intervals

Intervals can be chained together for complex date math:

```sql
SELECT now() + toIntervalDay(1) + toIntervalHour(6) AS tomorrow_morning_6am
```

## Summary

ClickHouse's `toInterval*()` family covers seconds through years and integrates cleanly with `+`/`-` operators on DateTime and Date values. Use them for rolling window filters, retention windows, and period comparisons. Both the `toIntervalDay()` style and the `INTERVAL n DAY` keyword syntax are supported and interchangeable.
