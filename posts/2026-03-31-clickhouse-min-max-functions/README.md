# How to Use min() and max() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Min, Max

Description: Learn how to use min() and max() in ClickHouse, including minIf/maxIf, DateTime comparisons, NULL behavior, and the argMin/argMax alternatives.

---

`min()` and `max()` return the smallest and largest values in a group. In ClickHouse they work across numeric, string, and date types, support conditional variants `minIf()` and `maxIf()`, and pair naturally with `argMin()` and `argMax()` when you need to retrieve a related column value alongside the extremes. This post covers all the patterns you will use day to day.

## Basic min() and max()

```sql
CREATE TABLE sensor_readings
(
    sensor_id   String,
    region      String,
    temperature Float64,
    recorded_at DateTime
)
ENGINE = MergeTree()
ORDER BY (recorded_at, sensor_id);

-- Global temperature range
SELECT
    min(temperature) AS min_temp,
    max(temperature) AS max_temp
FROM sensor_readings;

-- Per-sensor temperature range
SELECT
    sensor_id,
    min(temperature) AS min_temp,
    max(temperature) AS max_temp,
    max(temperature) - min(temperature) AS range_temp
FROM sensor_readings
GROUP BY sensor_id
ORDER BY range_temp DESC;
```

Both functions work on strings (lexicographic order) and dates as well:

```sql
-- Earliest and latest reading per sensor
SELECT
    sensor_id,
    min(recorded_at) AS first_reading,
    max(recorded_at) AS last_reading
FROM sensor_readings
GROUP BY sensor_id;
```

## NULL Behavior

`min()` and `max()` ignore NULL values. If all values in a group are NULL, the result is NULL.

```sql
CREATE TABLE nullable_demo
(
    id    UInt32,
    value Nullable(Float64)
)
ENGINE = MergeTree()
ORDER BY id;

INSERT INTO nullable_demo VALUES (1, 10.0), (2, NULL), (3, 5.0), (4, NULL);

-- NULLs are excluded: min = 5.0, max = 10.0
SELECT min(value), max(value) FROM nullable_demo;
```

To treat NULLs as a sentinel (e.g., treat NULL as 0 for min), use `ifNull()`:

```sql
SELECT min(ifNull(value, 0)) FROM nullable_demo;
```

## minIf() and maxIf() - Conditional Extremes

`minIf(column, condition)` and `maxIf(column, condition)` find the min or max only over rows where the condition is true. They complete in a single scan, making them ideal for side-by-side segment comparisons.

```sql
-- Compare temperature extremes between regions in one pass
SELECT
    minIf(temperature, region = 'north') AS north_min,
    maxIf(temperature, region = 'north') AS north_max,
    minIf(temperature, region = 'south') AS south_min,
    maxIf(temperature, region = 'south') AS south_max
FROM sensor_readings;
```

Use with time conditions to compare recent vs historical data:

```sql
SELECT
    sensor_id,
    maxIf(temperature, recorded_at >= now() - INTERVAL 1 HOUR) AS max_last_hour,
    max(temperature)                                             AS max_all_time
FROM sensor_readings
GROUP BY sensor_id;
```

## min() and max() with DateTime

DateTime comparisons are natural in ClickHouse - `min` and `max` return the chronologically earliest and latest timestamps.

```sql
-- Find the first and last event timestamps per day
SELECT
    toDate(recorded_at)   AS day,
    min(recorded_at)      AS first_event,
    max(recorded_at)      AS last_event,
    dateDiff('second', min(recorded_at), max(recorded_at)) AS span_seconds
FROM sensor_readings
GROUP BY day
ORDER BY day;
```

## argMin() and argMax() - Retrieve Associated Values

`min()` and `max()` return only the extreme value itself. When you need a different column from the row that contains the minimum or maximum, use `argMin(value, key)` and `argMax(value, key)`.

`argMin(val, key)` returns the value of `val` from the row where `key` is at its minimum.

```sql
-- Get the sensor_id that recorded the lowest temperature
SELECT argMin(sensor_id, temperature) AS coldest_sensor
FROM sensor_readings;

-- Get the timestamp when each sensor hit its peak temperature
SELECT
    sensor_id,
    max(temperature)                        AS peak_temp,
    argMax(recorded_at, temperature)        AS peak_time
FROM sensor_readings
GROUP BY sensor_id
ORDER BY peak_temp DESC;
```

### Getting the full row at the extreme

Combine `argMin`/`argMax` with multiple fields to reconstruct the "peak row":

```sql
-- Full details of the hottest reading per region
SELECT
    region,
    argMax(sensor_id,   temperature) AS hottest_sensor,
    argMax(recorded_at, temperature) AS hottest_at,
    max(temperature)                  AS peak_temp
FROM sensor_readings
GROUP BY region;
```

## Combining min/max in Aggregations

```sql
-- Daily statistics per sensor
SELECT
    sensor_id,
    toDate(recorded_at)          AS day,
    min(temperature)             AS daily_min,
    max(temperature)             AS daily_max,
    avg(temperature)             AS daily_avg,
    count()                       AS reading_count
FROM sensor_readings
GROUP BY sensor_id, day
ORDER BY sensor_id, day;
```

## Summary

`min()` and `max()` skip NULLs and work on numeric, string, and DateTime columns. Use `minIf()`/`maxIf()` to compute conditional extremes in a single scan. When you need a different column from the row that holds the extreme value, switch to `argMin(value, key)` or `argMax(value, key)` - they are the idiomatic ClickHouse way to answer "what other value was present when X was at its minimum or maximum?"
