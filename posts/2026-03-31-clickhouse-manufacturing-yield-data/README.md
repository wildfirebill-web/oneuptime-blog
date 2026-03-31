# How to Analyze Manufacturing Yield Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Manufacturing, Yield, Analytics, Quality, Production

Description: Analyze manufacturing yield across products, lines, and shifts in ClickHouse using rolled throughput yield, loss analysis, and trend monitoring.

---

## Overview

Yield analysis identifies where in a production process units are lost, what percentage pass without rework, and how yield varies by product, shift, and operator. ClickHouse's fast aggregations make it practical to analyze yield across millions of production records in seconds.

## Schema

```sql
CREATE TABLE yield_events
(
    batch_id       UInt64,
    product_code   LowCardinality(String),
    line_id        UInt16,
    station_id     UInt16,
    shift          LowCardinality(String),
    operator_id    UInt32,
    started_at     DateTime,
    input_units    UInt32,
    output_units   UInt32,
    scrap_units    UInt32,
    rework_units   UInt32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(started_at)
ORDER BY (line_id, station_id, started_at)
TTL started_at + INTERVAL 5 YEAR;
```

## Overall Yield by Product

```sql
SELECT
    product_code,
    toYYYYMM(started_at)                              AS month,
    sum(input_units)                                  AS input,
    sum(output_units)                                 AS output,
    sum(scrap_units)                                  AS scrap,
    sum(rework_units)                                 AS rework,
    round(sum(output_units) / sum(input_units) * 100, 2) AS yield_pct
FROM yield_events
WHERE started_at >= today() - INTERVAL 180 DAY
GROUP BY product_code, month
ORDER BY product_code, month;
```

## Rolled Throughput Yield (RTY)

RTY multiplies the first-pass yield at each station:

```sql
WITH station_fpy AS (
    SELECT
        product_code,
        station_id,
        toYYYYMM(started_at)                                     AS month,
        round(sum(output_units - rework_units) / sum(input_units), 6) AS fpy
    FROM yield_events
    WHERE started_at >= today() - INTERVAL 90 DAY
    GROUP BY product_code, station_id, month
)
SELECT
    product_code,
    month,
    round(exp(sum(log(fpy))) * 100, 3) AS rty_pct
FROM station_fpy
WHERE fpy > 0
GROUP BY product_code, month
ORDER BY product_code, month;
```

## Worst Performing Stations

```sql
SELECT
    line_id,
    station_id,
    toYYYYMM(started_at)                                   AS month,
    sum(scrap_units)                                       AS scrapped,
    sum(input_units)                                       AS total_in,
    round(sum(scrap_units) / sum(input_units) * 100, 2)   AS scrap_rate_pct
FROM yield_events
WHERE started_at >= today() - INTERVAL 90 DAY
GROUP BY line_id, station_id, month
ORDER BY scrap_rate_pct DESC
LIMIT 20;
```

## Yield Comparison by Shift

```sql
SELECT
    product_code,
    shift,
    round(avg(output_units) / avg(input_units) * 100, 2) AS avg_yield_pct,
    sum(scrap_units)    AS total_scrap,
    sum(rework_units)   AS total_rework
FROM yield_events
WHERE started_at >= today() - INTERVAL 30 DAY
GROUP BY product_code, shift
ORDER BY product_code, shift;
```

## Cost of Poor Quality (COPQ)

```sql
SELECT
    product_code,
    toYYYYMM(started_at) AS month,
    sum(scrap_units)     AS scrap_units,
    sum(rework_units)    AS rework_units,
    -- assume $50 scrap cost and $15 rework cost per unit
    round(sum(scrap_units) * 50 + sum(rework_units) * 15, 2) AS copq_usd
FROM yield_events
WHERE started_at >= today() - INTERVAL 365 DAY
GROUP BY product_code, month
ORDER BY copq_usd DESC;
```

## Yield Trend with Moving Average

```sql
SELECT
    line_id,
    toDate(started_at)                                        AS day,
    round(sum(output_units) / sum(input_units) * 100, 2)     AS daily_yield,
    round(avg(sum(output_units) / sum(input_units) * 100) OVER (
        PARTITION BY line_id
        ORDER BY toDate(started_at)
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS rolling_7d_yield
FROM yield_events
WHERE started_at >= today() - INTERVAL 60 DAY
GROUP BY line_id, day
ORDER BY line_id, day;
```

## Summary

ClickHouse enables comprehensive yield analysis with standard SQL, from simple yield percentages to rolled throughput yield across multi-station processes. Shift comparisons, COPQ calculations, and trend monitoring give quality and operations teams the data needed to drive continuous improvement programs effectively.
