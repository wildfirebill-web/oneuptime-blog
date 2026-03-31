# How to Use toIntervalDay() and Interval Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Interval, Date Arithmetic, Analytics

Description: Learn how toIntervalDay(), toIntervalMonth(), toIntervalYear() and related functions create Interval values for expressive and correct date arithmetic in ClickHouse.

---

ClickHouse provides a family of `toInterval*` functions that construct typed `Interval` values. These interval values can be added to or subtracted from `Date` and `DateTime` columns using the `+` and `-` operators, or passed to `dateAdd` and `dateSub`. The available constructors include `toIntervalSecond`, `toIntervalMinute`, `toIntervalHour`, `toIntervalDay`, `toIntervalWeek`, `toIntervalMonth`, `toIntervalQuarter`, and `toIntervalYear`. Using these functions makes date arithmetic readable and avoids the error-prone practice of converting dates to integers and doing manual multiplication.

## Basic Usage

```sql
-- Add intervals of different units to a base date
SELECT
    today() AS base_date,
    base_date + toIntervalDay(7)      AS one_week_later,
    base_date + toIntervalMonth(3)    AS three_months_later,
    base_date + toIntervalYear(1)     AS one_year_later,
    base_date - toIntervalDay(30)     AS thirty_days_ago;
```

```text
base_date   one_week_later  three_months_later  one_year_later  thirty_days_ago
2024-06-15  2024-06-22      2024-09-15          2025-06-15      2024-05-16
```

## Adding Intervals to DateTime Values

`toIntervalHour`, `toIntervalMinute`, and `toIntervalSecond` work with `DateTime` columns, preserving time-of-day information.

```sql
-- Compute expiry times from a base DateTime
SELECT
    now() AS created_at,
    created_at + toIntervalHour(1)    AS expires_in_1h,
    created_at + toIntervalMinute(30) AS expires_in_30m,
    created_at + toIntervalSecond(300) AS expires_in_5m;
```

## Using dateAdd and dateSub

`dateAdd` and `dateSub` accept an interval type keyword and an integer amount, providing a function-call syntax that some find more readable than the operator form.

```sql
-- dateAdd and dateSub syntax
SELECT
    event_time,
    dateAdd(hour,  2, event_time)  AS two_hours_later,
    dateSub(day,   1, event_time)  AS yesterday_same_time,
    dateAdd(month, 1, event_time)  AS next_month_same_time
FROM events
WHERE event_date = today()
LIMIT 5;
```

## Dynamic Intervals From a Column Value

The argument to `toInterval*` can be any integer expression, including a column value. This allows per-row interval computation.

```sql
-- Compute each subscription's expiry date based on its duration_months column
SELECT
    subscription_id,
    customer_id,
    start_date,
    duration_months,
    start_date + toIntervalMonth(duration_months) AS expiry_date
FROM subscriptions
WHERE status = 'active'
ORDER BY expiry_date;
```

## Combining Multiple Intervals

Intervals of different units can be added together in a single expression.

```sql
-- Advance a timestamp by 1 month, 2 days, and 6 hours simultaneously
SELECT
    now() AS base,
    base
        + toIntervalMonth(1)
        + toIntervalDay(2)
        + toIntervalHour(6) AS adjusted;
```

## Computing Rolling Window Boundaries

Intervals are particularly useful for defining the boundaries of rolling windows in WHERE clauses.

```sql
-- Events in the last 90 days
SELECT count()
FROM events
WHERE event_time >= now() - toIntervalDay(90);

-- Events in the last 3 months (calendar-accurate)
SELECT count()
FROM events
WHERE event_time >= now() - toIntervalMonth(3);
```

Note: `toIntervalDay(90)` and `toIntervalMonth(3)` are not equivalent because months have varying lengths. Choose based on whether calendar accuracy or fixed duration matters.

## Generating a Date Series With Intervals

You can combine `toIntervalDay` with `arrayJoin` or `numbers()` to generate a sequence of dates.

```sql
-- Generate the dates for the next 14 days
SELECT
    today() + toIntervalDay(number) AS future_date
FROM numbers(14)
ORDER BY future_date;
```

## Weekly Report Anchoring

To produce a weekly summary anchored to Monday boundaries, combine `toMonday` with `toIntervalWeek`.

```sql
-- Compare this week vs last week
SELECT
    toMonday(event_date) AS week_start,
    count() AS events
FROM events
WHERE
    event_date >= toMonday(today()) - toIntervalWeek(1)
    AND event_date < toMonday(today()) + toIntervalWeek(1)
GROUP BY week_start
ORDER BY week_start;
```

## Summary

The `toInterval*` family of functions constructs typed interval values that integrate naturally with ClickHouse date arithmetic using `+`, `-`, `dateAdd`, and `dateSub`. They accept integer expressions as arguments, enabling dynamic per-row interval computation. Use `toIntervalMonth` and `toIntervalYear` when calendar accuracy matters, and `toIntervalDay`/`toIntervalHour`/`toIntervalSecond` when you need fixed-duration offsets. Combining multiple interval types in a single expression is supported and produces intuitive results.
