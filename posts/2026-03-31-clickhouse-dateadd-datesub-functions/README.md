# How to Use dateAdd() and dateSub() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Date Arithmetic, Scheduling, Window Query

Description: Learn how dateAdd() and dateSub() add and subtract time intervals from DateTime values in ClickHouse for expiry dates, scheduling, and rolling window queries.

---

`dateAdd()` and `dateSub()` are the general-purpose date arithmetic functions in ClickHouse. They add or subtract a named time unit - such as days, months, or years - to a `Date` or `DateTime` value and return the same type. Together they cover every scenario where you need to shift a timestamp forward or backward by a calendar-aware amount, from computing trial expiry dates to building rolling 28-day retention windows.

## Function Signatures

```text
dateAdd(unit, n, dt)
dateSub(unit, n, dt)
```

`unit` is a string: `'second'`, `'minute'`, `'hour'`, `'day'`, `'week'`, `'month'`, `'quarter'`, `'year'`. `n` is a signed integer - passing a negative value to `dateAdd` subtracts, so the two functions are symmetric. The return type matches the input type.

## Basic Addition and Subtraction

The examples below show common interval shifts using the same base timestamp.

```sql
SELECT
    toDateTime('2026-03-31 14:00:00')                               AS base,
    dateAdd('second',  30,  toDateTime('2026-03-31 14:00:00'))      AS plus_30s,
    dateAdd('minute',  15,  toDateTime('2026-03-31 14:00:00'))      AS plus_15m,
    dateAdd('hour',    6,   toDateTime('2026-03-31 14:00:00'))      AS plus_6h,
    dateAdd('day',     7,   toDate('2026-03-31'))                   AS plus_1w,
    dateAdd('month',   3,   toDate('2026-03-31'))                   AS plus_3mo,
    dateAdd('year',    1,   toDate('2026-03-31'))                   AS plus_1yr,
    dateSub('day',     1,   toDate('2026-03-31'))                   AS yesterday;
```

## Computing Trial and Subscription Expiry Dates

When a user starts a free trial, you can store the signup date and compute the expiry at query time - or compute it at write time for efficient range queries.

```sql
-- Compute expiry at query time
SELECT
    user_id,
    signup_date,
    dateAdd('day', 14, signup_date)   AS trial_expiry,
    dateAdd('month', 1, signup_date)  AS monthly_renewal,
    dateAdd('year',  1, signup_date)  AS annual_renewal
FROM users
WHERE plan = 'trial'
ORDER BY trial_expiry ASC;
```

Flag users whose trial expires within the next 48 hours for re-engagement emails:

```sql
SELECT
    user_id,
    email,
    dateAdd('day', 14, signup_date) AS trial_expiry
FROM users
WHERE plan = 'trial'
  AND trial_expiry BETWEEN now() AND dateAdd('hour', 48, now())
ORDER BY trial_expiry ASC;
```

## Rolling Window Queries

Rolling windows compare recent activity against a past window of equal length. The query below counts orders in the current 30-day window vs the prior 30-day window.

```sql
SELECT
    count(CASE WHEN order_date >= dateSub('day', 30, today()) THEN 1 END) AS last_30d,
    count(CASE WHEN order_date >= dateSub('day', 60, today())
               AND order_date <  dateSub('day', 30, today()) THEN 1 END)  AS prior_30d
FROM orders
WHERE order_date >= dateSub('day', 60, today());
```

## Scheduling Future Events

Job schedulers and notification systems need to compute future trigger times. `dateAdd` works directly in INSERT statements.

```sql
INSERT INTO scheduled_jobs (job_id, created_at, run_at, job_type)
SELECT
    job_id,
    now()                              AS created_at,
    dateAdd('hour', retry_delay_hours, now()) AS run_at,
    job_type
FROM failed_jobs
WHERE retry_count < 3;
```

## Month-End Anchor Calculations

`dateAdd('month', n, dt)` is calendar-aware and handles varying month lengths correctly. Adding one month to January 31 gives February 28 (or 29 in a leap year) rather than an invalid date.

```sql
SELECT
    toDate('2026-01-31')                       AS jan_31,
    dateAdd('month', 1, toDate('2026-01-31'))   AS feb_result,   -- 2026-02-28
    dateAdd('month', 2, toDate('2026-01-31'))   AS mar_result,   -- 2026-03-31
    dateAdd('month', 1, toDate('2026-02-28'))   AS mar_from_feb; -- 2026-03-28
```

## Building a Date Range Table

`dateAdd` combined with `arrayJoin` and `range()` generates a series of dates - useful for filling gaps in time series results.

```sql
SELECT
    dateAdd('day', number, toDate('2026-03-01')) AS day
FROM (
    SELECT arrayJoin(range(0, 30)) AS number
)
ORDER BY day ASC;
```

## Summary

`dateAdd()` and `dateSub()` are the general interval arithmetic functions in ClickHouse, complementing the shorthand `addDays`, `addMonths`, and `addYears` functions. They are calendar-aware, handle month-end edge cases correctly, and work with both `Date` and `DateTime` types. Use them for expiry calculations, scheduling, rolling window boundaries, and any query where you need to shift a timestamp by a configurable interval.
