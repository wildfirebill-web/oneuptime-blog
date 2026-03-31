# How to Use toInterval Functions for Type Conversion in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Type Conversion, Interval, Date Arithmetic, DateTime

Description: Learn how to use toIntervalSecond(), toIntervalDay(), toIntervalMonth() and other toInterval functions to create Interval values for date arithmetic.

---

ClickHouse provides a set of `toInterval*` functions that convert an integer to an `Interval` type. Interval values are used in date and time arithmetic expressions. Rather than writing `INTERVAL 7 DAY` as a literal, you can use `toIntervalDay(7)` which is more dynamic and can accept a column or computed value as the duration.

## Available toInterval Functions

```text
toIntervalSecond(n)   -> IntervalSecond
toIntervalMinute(n)   -> IntervalMinute
toIntervalHour(n)     -> IntervalHour
toIntervalDay(n)      -> IntervalDay
toIntervalWeek(n)     -> IntervalWeek
toIntervalMonth(n)    -> IntervalMonth
toIntervalQuarter(n)  -> IntervalQuarter
toIntervalYear(n)     -> IntervalYear
```

## Basic Usage

```sql
-- Create interval values from integers
SELECT toIntervalDay(7)      AS one_week;
SELECT toIntervalMonth(3)    AS one_quarter;
SELECT toIntervalSecond(300) AS five_minutes;

-- Check the type
SELECT toTypeName(toIntervalDay(7)) AS interval_type;
-- Returns: 'IntervalDay'
```

## Date Arithmetic with toInterval

The primary use case is adding or subtracting a computed number of units from a date or datetime.

```sql
-- Add a dynamic number of days to a date
SELECT
    user_id,
    signup_date,
    trial_days,
    signup_date + toIntervalDay(trial_days) AS trial_end_date
FROM subscriptions
LIMIT 10;

-- Subtract a variable number of hours from a datetime
SELECT
    event_id,
    event_time,
    lookback_hours,
    event_time - toIntervalHour(lookback_hours) AS window_start
FROM events
LIMIT 10;
```

## Compared to INTERVAL Literal Syntax

```sql
-- Literal interval (fixed duration)
SELECT today() + INTERVAL 7 DAY AS next_week;

-- toIntervalDay with a variable value
SELECT
    user_id,
    today() + toIntervalDay(days_until_renewal) AS renewal_date
FROM user_subscriptions
LIMIT 10;
```

The `INTERVAL n UNIT` syntax requires a literal integer. `toIntervalDay(n)` accepts any integer expression including column values.

## Computing Retention Windows

```sql
-- Compute a rolling N-day retention window per user
SELECT
    user_id,
    cohort_date,
    retention_days,
    cohort_date + toIntervalDay(retention_days) AS retention_cutoff
FROM cohort_analysis
LIMIT 10;
```

## Expiry Date Calculation

```sql
-- Compute subscription expiry dates with variable durations
SELECT
    subscription_id,
    start_date,
    duration_months,
    start_date + toIntervalMonth(duration_months) AS expiry_date
FROM subscriptions
ORDER BY expiry_date
LIMIT 20;

-- Find subscriptions expiring in the next 30 days
SELECT
    subscription_id,
    user_id,
    expiry_date
FROM (
    SELECT
        subscription_id,
        user_id,
        start_date + toIntervalMonth(duration_months) AS expiry_date
    FROM subscriptions
)
WHERE expiry_date BETWEEN today() AND today() + toIntervalDay(30)
ORDER BY expiry_date;
```

## toIntervalYear and toIntervalQuarter for Financial Periods

```sql
-- Add a number of years to a fiscal start date
SELECT
    fiscal_start,
    years_to_add,
    fiscal_start + toIntervalYear(years_to_add) AS future_fiscal_start
FROM fiscal_calendar
LIMIT 10;

-- Compute the start of the next N quarters
SELECT
    quarter_start,
    n,
    quarter_start + toIntervalQuarter(n) AS nth_quarter_start
FROM (
    SELECT toStartOfQuarter(today()) AS quarter_start, number + 1 AS n
    FROM numbers(4)
);
```

## Negative Intervals (Subtraction)

```sql
-- Use negative values for subtraction
SELECT
    event_time,
    event_time + toIntervalHour(-24) AS yesterday_same_time,
    event_time - toIntervalHour(24)  AS equivalent_subtraction
FROM events
LIMIT 5;
```

## Dynamic Aggregation Window

```sql
-- Count events in a dynamic time window per record
SELECT
    e.event_id,
    e.event_time,
    e.window_seconds,
    count(e2.event_id) AS events_in_window
FROM events AS e
JOIN events AS e2
    ON e2.user_id = e.user_id
    AND e2.event_time BETWEEN e.event_time
        AND e.event_time + toIntervalSecond(e.window_seconds)
GROUP BY e.event_id, e.event_time, e.window_seconds
LIMIT 10;
```

## Summary

The `toInterval*` functions convert integer values to ClickHouse `Interval` types that can be added to or subtracted from `Date` and `DateTime` values. They are the dynamic alternative to `INTERVAL n UNIT` literal syntax, accepting column values and computed expressions as the duration. Use them for subscription expiry calculations, rolling window analysis, and any date arithmetic where the interval duration varies per row.
