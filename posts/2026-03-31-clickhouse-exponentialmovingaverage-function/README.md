# How to Use exponentialMovingAverage() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, EMA, Moving Average, Time Series

Description: Learn how to use exponentialMovingAverage() in ClickHouse to compute time-weighted EMA with a configurable alpha decay factor for time series smoothing.

---

Exponential Moving Average (EMA) is a type of weighted moving average where more recent values carry exponentially more weight than older ones. ClickHouse's `exponentialMovingAverage()` aggregate function computes a time-weighted EMA using a user-defined alpha (decay) parameter, making it suitable for smoothing noisy time series data, tracking trends, and building real-time dashboards directly in SQL.

## What Is EMA and the Alpha Parameter

EMA applies a decay factor so that each new value has a weight of `alpha`, and each older value has its weight multiplied by `(1 - alpha)` for each time step it ages:

```text
EMA(t) = alpha * value(t) + (1 - alpha) * EMA(t-1)
```

The `alpha` parameter controls smoothing:
- **Alpha close to 1.0** - fast-responding, tracks recent values closely (less smoothing)
- **Alpha close to 0.0** - slow-responding, heavily smoothed (lags behind changes)
- **Common values** - 0.1 to 0.3 for moderate smoothing

ClickHouse's implementation is time-weighted, meaning it adjusts for gaps between timestamps, not just the count of data points.

## Syntax

```sql
exponentialMovingAverage(alpha)(value, timestamp)
```

Parameters:
- `alpha` - decay factor in (0, 1], a `Float64` constant
- `value` - numeric column of measurements
- `timestamp` - time column (DateTime, Unix timestamp, or any numeric)

Returns a `Float64` representing the EMA at the last observed timestamp.

## Creating Sample Data

```sql
CREATE TABLE sensor_readings
(
    sensor_id UInt32,
    ts        DateTime,
    reading   Float64
)
ENGINE = MergeTree()
ORDER BY (sensor_id, ts);

INSERT INTO sensor_readings VALUES
    (1, '2024-01-01 00:00:00', 10.0),
    (1, '2024-01-01 00:01:00', 12.0),
    (1, '2024-01-01 00:02:00',  9.5),
    (1, '2024-01-01 00:03:00', 11.0),
    (1, '2024-01-01 00:04:00', 13.5),
    (1, '2024-01-01 00:05:00', 10.5),
    (1, '2024-01-01 00:06:00', 14.0),
    (1, '2024-01-01 00:07:00', 11.5),
    (1, '2024-01-01 00:08:00', 12.5),
    (1, '2024-01-01 00:09:00', 15.0);
```

## Basic EMA Calculation

```sql
SELECT exponentialMovingAverage(0.2)(reading, toUnixTimestamp(ts))
FROM sensor_readings
WHERE sensor_id = 1;
```

## Comparing Different Alpha Values

```sql
SELECT
    exponentialMovingAverage(0.1)(reading, toUnixTimestamp(ts)) AS ema_slow,
    exponentialMovingAverage(0.3)(reading, toUnixTimestamp(ts)) AS ema_medium,
    exponentialMovingAverage(0.7)(reading, toUnixTimestamp(ts)) AS ema_fast,
    avg(reading)                                                 AS simple_average
FROM sensor_readings
WHERE sensor_id = 1;
```

Higher alpha values produce a faster EMA that closely tracks recent fluctuations; lower values produce a smoother result.

## Per-Sensor EMA

```sql
SELECT
    sensor_id,
    exponentialMovingAverage(0.2)(reading, toUnixTimestamp(ts)) AS ema
FROM sensor_readings
GROUP BY sensor_id
ORDER BY sensor_id;
```

## Windowed EMA for Trend Analysis

Use `GROUP BY` on time windows to compute the EMA for each hour:

```sql
SELECT
    toStartOfHour(ts)                                                   AS hour,
    exponentialMovingAverage(0.3)(reading, toUnixTimestamp(ts))         AS hourly_ema,
    avg(reading)                                                        AS hourly_avg
FROM sensor_readings
GROUP BY hour
ORDER BY hour;
```

## Revenue EMA for Dashboard Smoothing

```sql
CREATE TABLE daily_revenue
(
    day     Date,
    revenue Float64
)
ENGINE = MergeTree()
ORDER BY day;

INSERT INTO daily_revenue VALUES
    ('2024-01-01', 1000.0),
    ('2024-01-02', 1200.0),
    ('2024-01-03',  950.0),
    ('2024-01-04', 1100.0),
    ('2024-01-05', 1350.0),
    ('2024-01-06', 1050.0),
    ('2024-01-07', 1400.0);

SELECT
    exponentialMovingAverage(0.3)(revenue, toUnixTimestamp(toDateTime(day))) AS revenue_ema
FROM daily_revenue;
```

## Handling Irregular Timestamps

Because ClickHouse's EMA is time-weighted, it automatically adjusts for uneven gaps between readings. A gap of 10 minutes between two readings counts differently from a gap of 1 minute:

```sql
-- Irregular intervals are handled naturally
INSERT INTO sensor_readings VALUES
    (2, '2024-01-01 00:00:00', 10.0),
    (2, '2024-01-01 00:05:00', 12.0),  -- 5-minute gap
    (2, '2024-01-01 00:06:00',  9.5),  -- 1-minute gap
    (2, '2024-01-01 00:20:00', 11.0);  -- 14-minute gap

SELECT exponentialMovingAverage(0.2)(reading, toUnixTimestamp(ts))
FROM sensor_readings
WHERE sensor_id = 2;
```

## EMA with SimpleAggregateFunction in AggregatingMergeTree

For incremental EMA in materialized views, store partial state using `AggregatingMergeTree`:

```sql
CREATE TABLE sensor_ema_mv
(
    sensor_id UInt32,
    ema_state AggregateFunction(exponentialMovingAverage(0.2), Float64, UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY sensor_id;

CREATE MATERIALIZED VIEW sensor_ema_view TO sensor_ema_mv AS
SELECT
    sensor_id,
    exponentialMovingAverageState(0.2)(reading, toUnixTimestamp(ts)) AS ema_state
FROM sensor_readings
GROUP BY sensor_id;

-- Query the materialized view
SELECT
    sensor_id,
    exponentialMovingAverageMerge(0.2)(ema_state) AS ema
FROM sensor_ema_mv
GROUP BY sensor_id;
```

## Summary

`exponentialMovingAverage()` in ClickHouse computes a time-weighted EMA using a configurable alpha decay factor. It handles irregular timestamps naturally by weighting based on actual time differences, not row counts. Use it for smoothing sensor data, tracking revenue trends, building dashboards, and detecting drift in time series metrics. Adjust alpha to control the trade-off between responsiveness and smoothness, and combine with `AggregatingMergeTree` for incremental computation at scale.
