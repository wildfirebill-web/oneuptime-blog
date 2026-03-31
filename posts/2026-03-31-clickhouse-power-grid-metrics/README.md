# How to Track Power Grid Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Power Grid, Energy, SCADA, Analytics, Reliability

Description: Track power grid operational metrics in ClickHouse - frequency deviations, voltage stability, outage events, and generation-demand balance.

---

Power grid operators require real-time and historical analytics on grid frequency, voltage, generation mix, and reliability events. ClickHouse handles the high-frequency SCADA data that drives these analyses.

## Grid Measurements Table

```sql
CREATE TABLE grid_measurements (
    substation_id  LowCardinality(String),
    region         LowCardinality(String),
    measurement    LowCardinality(String),  -- frequency_hz, voltage_kv, load_mw, generation_mw
    value          Float64,
    quality_flag   LowCardinality(String),  -- good, suspect, bad
    recorded_at    DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(recorded_at)
ORDER BY (substation_id, measurement, recorded_at);
```

## Grid Frequency Stability

Track deviation from nominal 50/60 Hz:

```sql
SELECT
    region,
    toStartOfMinute(recorded_at) AS minute,
    avg(value) AS avg_freq_hz,
    min(value) AS min_freq_hz,
    max(value) AS max_freq_hz,
    countIf(abs(value - 50.0) > 0.2) AS deviation_events
FROM grid_measurements
WHERE measurement = 'frequency_hz'
  AND quality_flag = 'good'
  AND recorded_at >= now() - INTERVAL 24 HOUR
GROUP BY region, minute
ORDER BY region, minute;
```

## Generation vs. Demand Balance

```sql
SELECT
    region,
    toStartOfHour(recorded_at) AS hour,
    sumIf(value, measurement = 'generation_mw') AS total_generation_mw,
    sumIf(value, measurement = 'load_mw') AS total_load_mw,
    sumIf(value, measurement = 'generation_mw') - sumIf(value, measurement = 'load_mw') AS balance_mw
FROM grid_measurements
WHERE quality_flag = 'good'
  AND recorded_at >= today() - 7
GROUP BY region, hour
ORDER BY region, hour;
```

## Voltage Excursion Events

```sql
SELECT
    substation_id,
    region,
    recorded_at,
    value AS voltage_kv,
    multiIf(
        value < 345 * 0.95, 'under_voltage',
        value > 345 * 1.05, 'over_voltage',
        'normal'
    ) AS voltage_status
FROM grid_measurements
WHERE measurement = 'voltage_kv'
  AND abs(value - 345) / 345 > 0.05
  AND quality_flag = 'good'
  AND recorded_at >= today() - 7
ORDER BY recorded_at DESC
LIMIT 200;
```

## Outage Event Summary

```sql
CREATE TABLE grid_outages (
    outage_id     String,
    substation_id LowCardinality(String),
    region        LowCardinality(String),
    cause         LowCardinality(String),
    customers_affected UInt32,
    start_at      DateTime,
    restored_at   Nullable(DateTime),
    duration_min  Nullable(UInt32)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(start_at)
ORDER BY (region, start_at);
```

```sql
SELECT
    region,
    toStartOfMonth(start_at) AS month,
    count() AS outage_count,
    sum(customers_affected) AS total_customers_affected,
    avg(duration_min) AS avg_duration_min,
    sum(customers_affected * duration_min) / 60 AS customer_hours_interrupted
FROM grid_outages
WHERE start_at >= today() - 365
GROUP BY region, month
ORDER BY region, month;
```

## Renewable Generation Share

```sql
SELECT
    region,
    toDate(recorded_at) AS day,
    sumIf(value, measurement = 'solar_mw') AS solar_mw,
    sumIf(value, measurement = 'wind_mw') AS wind_mw,
    sumIf(value, measurement = 'generation_mw') AS total_mw,
    round((sumIf(value, measurement = 'solar_mw') + sumIf(value, measurement = 'wind_mw'))
        / nullIf(sumIf(value, measurement = 'generation_mw'), 0) * 100, 2) AS renewable_pct
FROM grid_measurements
WHERE recorded_at >= today() - 30
  AND quality_flag = 'good'
GROUP BY region, day
ORDER BY region, day;
```

## Summary

ClickHouse provides a scalable platform for power grid analytics - from real-time frequency monitoring and voltage excursion detection to outage tracking and renewable generation dashboards. High-frequency SCADA data is stored efficiently and queried rapidly with time-series partitioning.
