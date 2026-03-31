# How to Store Smart Meter Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Smart Meter, IoT, Energy, Time Series, Utility

Description: Store and query smart meter data in ClickHouse with efficient schema design for high-frequency electricity, gas, and water readings at scale.

---

Smart meters generate readings every 15-30 minutes per meter. A utility with 1 million meters produces 50+ million rows per day. ClickHouse is an ideal store for this data with its columnar compression and fast time-series queries.

## Smart Meter Readings Table

```sql
CREATE TABLE smart_meter_readings (
    meter_id       LowCardinality(String),
    customer_id    UInt64,
    account_id     String,
    meter_type     LowCardinality(String),  -- electricity, gas, water
    tariff_code    LowCardinality(String),
    reading        Float64,  -- cumulative kWh, cubic meters, etc.
    consumption    Float32,  -- delta from previous reading
    reading_quality LowCardinality(String),  -- actual, estimated, missing
    recorded_at    DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (meter_id, recorded_at)
SETTINGS index_granularity = 8192;
```

## Daily Consumption Per Meter

```sql
SELECT
    meter_id,
    toDate(recorded_at) AS reading_date,
    sum(consumption) AS daily_kwh,
    countIf(reading_quality = 'actual') AS actual_readings,
    countIf(reading_quality = 'estimated') AS estimated_readings
FROM smart_meter_readings
WHERE recorded_at >= today() - 30
  AND meter_type = 'electricity'
GROUP BY meter_id, reading_date
ORDER BY meter_id, reading_date;
```

## Hourly Load Profile

Compute average demand at each hour of day:

```sql
SELECT
    toHour(recorded_at) AS hour_of_day,
    avg(consumption) AS avg_kwh,
    quantile(0.95)(consumption) AS p95_kwh,
    sum(consumption) AS total_kwh
FROM smart_meter_readings
WHERE recorded_at >= today() - 30
  AND meter_type = 'electricity'
  AND reading_quality = 'actual'
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

## Anomaly Detection - Unusually High Readings

Flag meters with consumption well above their historical baseline:

```sql
SELECT
    meter_id,
    toDate(recorded_at) AS reading_date,
    sum(consumption) AS daily_consumption,
    avg(sum(consumption)) OVER (
        PARTITION BY meter_id
        ORDER BY toDate(recorded_at)
        ROWS BETWEEN 29 PRECEDING AND 1 PRECEDING
    ) AS rolling_30d_avg,
    sum(consumption) / nullIf(avg(sum(consumption)) OVER (
        PARTITION BY meter_id ORDER BY toDate(recorded_at)
        ROWS BETWEEN 29 PRECEDING AND 1 PRECEDING
    ), 0) AS consumption_ratio
FROM smart_meter_readings
WHERE recorded_at >= today() - 60
GROUP BY meter_id, reading_date
HAVING consumption_ratio > 3
ORDER BY consumption_ratio DESC;
```

## Data Quality Report

Track the percentage of estimated vs. actual readings per region:

```sql
SELECT
    toStartOfWeek(recorded_at) AS week,
    meter_type,
    countIf(reading_quality = 'actual') AS actual_count,
    countIf(reading_quality = 'estimated') AS estimated_count,
    countIf(reading_quality = 'missing') AS missing_count,
    count() AS total,
    round(countIf(reading_quality = 'actual') / count() * 100, 2) AS actual_pct
FROM smart_meter_readings
WHERE recorded_at >= today() - 90
GROUP BY week, meter_type
ORDER BY week, meter_type;
```

## Peak vs. Off-Peak Split

For time-of-use tariff billing:

```sql
SELECT
    meter_id,
    toYYYYMM(recorded_at) AS bill_month,
    sumIf(consumption, toHour(recorded_at) BETWEEN 7 AND 22) AS peak_kwh,
    sumIf(consumption, toHour(recorded_at) NOT BETWEEN 7 AND 22) AS offpeak_kwh,
    peak_kwh + offpeak_kwh AS total_kwh
FROM smart_meter_readings
WHERE recorded_at >= today() - 90
  AND meter_type = 'electricity'
  AND reading_quality = 'actual'
GROUP BY meter_id, bill_month
ORDER BY meter_id, bill_month;
```

## Summary

ClickHouse is purpose-built for smart meter data storage - columnar compression reduces storage costs dramatically for repetitive meter readings, while fast aggregations enable daily consumption reports, anomaly detection, and billing calculations across millions of meters.
