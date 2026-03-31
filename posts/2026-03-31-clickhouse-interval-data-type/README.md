# How to Use Interval Data Type in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Type, Interval, Date, DateTime

Description: Learn how to use ClickHouse's Interval type for date and time arithmetic with INTERVAL syntax and toInterval functions.

---

ClickHouse's `Interval` data type represents a duration of time - such as 3 days, 2 months, or 1 year - that can be added to or subtracted from `Date`, `Date32`, `DateTime`, or `DateTime64` values. Instead of computing offsets in raw seconds or days, `Interval` lets you express time arithmetic naturally and readably. This post covers the full range of interval units, both the `INTERVAL` keyword syntax and the `toIntervalX()` function family, and walks through practical examples.

## Interval Units Supported

ClickHouse supports interval values for the following units:

| Keyword Syntax | Function Equivalent | Description |
|---|---|---|
| `INTERVAL N NANOSECOND` | `toIntervalNanosecond(N)` | Nanoseconds (DateTime64 only) |
| `INTERVAL N MICROSECOND` | `toIntervalMicrosecond(N)` | Microseconds (DateTime64 only) |
| `INTERVAL N MILLISECOND` | `toIntervalMillisecond(N)` | Milliseconds (DateTime64 only) |
| `INTERVAL N SECOND` | `toIntervalSecond(N)` | Seconds |
| `INTERVAL N MINUTE` | `toIntervalMinute(N)` | Minutes |
| `INTERVAL N HOUR` | `toIntervalHour(N)` | Hours |
| `INTERVAL N DAY` | `toIntervalDay(N)` | Days |
| `INTERVAL N WEEK` | `toIntervalWeek(N)` | Weeks |
| `INTERVAL N MONTH` | `toIntervalMonth(N)` | Months |
| `INTERVAL N QUARTER` | `toIntervalQuarter(N)` | Quarters |
| `INTERVAL N YEAR` | `toIntervalYear(N)` | Years |

## Basic Interval Arithmetic with Dates

Adding and subtracting intervals from `Date` values is the most common use case:

```sql
SELECT
    today()                              AS today,
    today() + INTERVAL 7 DAY             AS next_week,
    today() - INTERVAL 1 MONTH          AS last_month,
    today() + INTERVAL 1 YEAR           AS next_year,
    today() - INTERVAL 3 MONTH          AS three_months_ago,
    today() + INTERVAL 2 WEEK           AS two_weeks_from_now;
```

## Interval Arithmetic with DateTime

For `DateTime` and `DateTime64`, you can use sub-day intervals for precise time calculations:

```sql
SELECT
    now()                                AS current_time,
    now() + INTERVAL 1 HOUR             AS one_hour_later,
    now() - INTERVAL 30 MINUTE          AS thirty_minutes_ago,
    now() + INTERVAL 90 SECOND          AS ninety_seconds_later,
    now() - INTERVAL 2 DAY              AS two_days_ago;
```

With `DateTime64` for nanosecond/microsecond precision:

```sql
SELECT
    now64(9)                                       AS now_ns,
    now64(9) + INTERVAL 500 MILLISECOND            AS half_second_later,
    now64(9) - INTERVAL 1000 MICROSECOND           AS one_ms_ago,
    now64(9) + INTERVAL 1000000 NANOSECOND         AS one_ms_later;
```

## Using toIntervalX() Functions

The `toIntervalX()` function family is useful when the interval value comes from a column or expression rather than a literal:

```sql
-- Using a column value as the interval amount
SELECT
    event_date + toIntervalDay(retention_days)  AS expiry_date,
    event_date - toIntervalMonth(lookback)      AS lookback_start
FROM (
    SELECT
        today()   AS event_date,
        30        AS retention_days,
        3         AS lookback
);
```

Dynamic intervals based on computed values:

```sql
SELECT
    toDate('2026-01-01') + toIntervalMonth(month_offset) AS month_start
FROM (
    SELECT arrayJoin(range(0, 12)) AS month_offset
);
```

## Practical Example - Time Window Queries

Intervals are commonly used to define rolling time windows in analytical queries:

```sql
-- Events in the last 24 hours
SELECT
    count()    AS event_count,
    toHour(event_time) AS hour_of_day
FROM events
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

```sql
-- Week-over-week comparison
SELECT
    toDate(event_time)                      AS event_date,
    countIf(event_time >= now() - INTERVAL 7 DAY
        AND event_time < now())             AS current_week,
    countIf(event_time >= now() - INTERVAL 14 DAY
        AND event_time < now() - INTERVAL 7 DAY) AS prior_week
FROM events
WHERE event_time >= now() - INTERVAL 14 DAY
GROUP BY event_date
ORDER BY event_date;
```

## Combining Multiple Intervals

ClickHouse does not support directly adding multiple interval types together in a single expression, but you can chain additions:

```sql
SELECT
    toDate('2026-01-01')
        + INTERVAL 1 YEAR
        + INTERVAL 2 MONTH
        + INTERVAL 15 DAY  AS result_date;
```

```sql
SELECT
    now()
        + INTERVAL 2 HOUR
        + INTERVAL 30 MINUTE  AS result_time;
```

## Interval in Table Definitions with TTL

Intervals are used in `TTL` expressions to specify data expiry or tiering policies:

```sql
CREATE TABLE telemetry_events
(
    event_time  DateTime,
    service     String,
    metric      Float64
)
ENGINE = MergeTree()
ORDER BY (service, event_time)
TTL event_time + INTERVAL 90 DAY;
```

Moving data to cold storage after 30 days:

```sql
CREATE TABLE telemetry_tiered
(
    event_time  DateTime,
    service     String,
    metric      Float64
)
ENGINE = MergeTree()
ORDER BY (service, event_time)
TTL
    event_time + INTERVAL 30 DAY  TO DISK 'cold_disk',
    event_time + INTERVAL 365 DAY DELETE;
```

## Interval Type Inspection

You can inspect the type of an interval expression using `toTypeName`:

```sql
SELECT
    toTypeName(INTERVAL 1 DAY)     AS day_type,
    toTypeName(INTERVAL 3 MONTH)   AS month_type,
    toTypeName(INTERVAL 2 HOUR)    AS hour_type,
    toTypeName(toIntervalYear(5))  AS year_func_type;
-- Results: IntervalDay, IntervalMonth, IntervalHour, IntervalYear
```

Note that different interval units are distinct types - `IntervalDay` and `IntervalMonth` cannot be directly mixed in a single expression without chaining.

## Summary

ClickHouse's `Interval` type makes date and time arithmetic expressive and readable. Use the `INTERVAL N UNIT` syntax for literals and the `toIntervalX()` functions when interval amounts come from columns or expressions. Intervals are essential for rolling time windows, TTL policies, and dynamic date calculations across the full range of units from nanoseconds to years.
