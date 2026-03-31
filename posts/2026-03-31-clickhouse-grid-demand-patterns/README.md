# How to Analyze Grid Demand Patterns with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Grid Demand, Energy, Time Series, Load Forecasting, Analytics

Description: Analyze electricity grid demand patterns in ClickHouse to identify peak loads, seasonal trends, day-type profiles, and demand response opportunities.

---

Understanding grid demand patterns helps utilities plan capacity, procure power, and design demand response programs. ClickHouse can aggregate interval-level demand data across the grid to reveal load shapes and seasonal behaviors.

## Grid Demand Table

```sql
CREATE TABLE grid_demand (
    region_id    LowCardinality(String),
    substation_id LowCardinality(String),
    feeder_id    LowCardinality(String),
    demand_mw    Float32,
    temperature_c Float32,
    is_holiday   UInt8,
    recorded_at  DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (region_id, recorded_at);
```

## Average Hourly Load Shape

Compute the typical hourly demand profile for weekdays vs. weekends:

```sql
SELECT
    region_id,
    toHour(recorded_at) AS hour,
    toDayOfWeek(recorded_at) IN (6, 7) AS is_weekend,
    avg(demand_mw) AS avg_demand_mw,
    quantile(0.95)(demand_mw) AS p95_demand_mw
FROM grid_demand
WHERE recorded_at >= today() - 90
GROUP BY region_id, hour, is_weekend
ORDER BY region_id, is_weekend, hour;
```

## Monthly Peak Demand

Track monthly system peak for capacity planning:

```sql
SELECT
    region_id,
    toStartOfMonth(recorded_at) AS month,
    max(demand_mw) AS monthly_peak_mw,
    argMax(recorded_at, demand_mw) AS peak_timestamp,
    avg(demand_mw) AS avg_demand_mw,
    round(avg(demand_mw) / max(demand_mw) * 100, 2) AS load_factor_pct
FROM grid_demand
WHERE recorded_at >= today() - 365
GROUP BY region_id, month
ORDER BY region_id, month;
```

## Temperature Sensitivity Analysis

Understand how demand responds to temperature changes:

```sql
SELECT
    region_id,
    round(temperature_c / 5) * 5 AS temp_bucket,
    avg(demand_mw) AS avg_demand_mw,
    count() AS observations
FROM grid_demand
WHERE recorded_at >= today() - 365
  AND toHour(recorded_at) BETWEEN 14 AND 18  -- afternoon peak period
GROUP BY region_id, temp_bucket
ORDER BY region_id, temp_bucket;
```

## Demand Response Event Analysis

Measure load reduction during demand response events:

```sql
SELECT
    dr_event_id,
    region_id,
    event_start,
    event_end,
    avg_baseline_mw,
    avg_actual_mw,
    avg_baseline_mw - avg_actual_mw AS load_reduction_mw,
    round((avg_baseline_mw - avg_actual_mw) / avg_baseline_mw * 100, 2) AS reduction_pct
FROM (
    SELECT
        region_id,
        'DR_EVENT_001' AS dr_event_id,
        toDateTime('2026-01-15 14:00:00') AS event_start,
        toDateTime('2026-01-15 17:00:00') AS event_end,
        avgIf(demand_mw, recorded_at BETWEEN event_start - INTERVAL 1 WEEK
            AND event_start - INTERVAL 1 WEEK + INTERVAL 3 HOUR) AS avg_baseline_mw,
        avgIf(demand_mw, recorded_at BETWEEN event_start AND event_end) AS avg_actual_mw
    FROM grid_demand
    WHERE region_id = 'NORTH'
      AND recorded_at >= toDateTime('2026-01-08 14:00:00')
    GROUP BY region_id
);
```

## Holiday vs. Workday Demand Comparison

```sql
SELECT
    region_id,
    is_holiday,
    toHour(recorded_at) AS hour,
    avg(demand_mw) AS avg_demand_mw
FROM grid_demand
WHERE recorded_at >= today() - 365
GROUP BY region_id, is_holiday, hour
ORDER BY region_id, is_holiday, hour;
```

## Summary

ClickHouse enables deep analysis of grid demand patterns - load shapes, seasonal trends, temperature sensitivity, and demand response effectiveness are all computable from interval data. These insights inform capacity planning, procurement, and demand management programs.
