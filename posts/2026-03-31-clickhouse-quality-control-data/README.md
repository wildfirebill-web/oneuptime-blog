# How to Analyze Quality Control Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Quality Control, Manufacturing, Analytics, SPC, Defect

Description: Analyze manufacturing quality control data in ClickHouse with statistical process control queries, defect Pareto analysis, and trend detection.

---

## Overview

Quality control analytics requires detecting process drift before it produces defects, identifying the most common failure modes, and tracking yield trends across shifts and lines. ClickHouse's aggregate functions and window functions support these SPC (Statistical Process Control) patterns efficiently.

## Schema

```sql
CREATE TABLE qc_measurements
(
    part_id        UInt64,
    line_id        UInt16,
    station_id     UInt16,
    inspector_id   UInt16,
    measured_at    DateTime,
    dimension_id   UInt8,
    measured_value Float32,
    nominal_value  Float32,
    tolerance_plus Float32,
    tolerance_minus Float32,
    pass           UInt8
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(measured_at)
ORDER BY (line_id, station_id, measured_at)
TTL measured_at + INTERVAL 5 YEAR;

CREATE TABLE defect_log
(
    part_id        UInt64,
    line_id        UInt16,
    defect_code    UInt16,
    defect_desc    String,
    detected_at    DateTime,
    severity       UInt8
)
ENGINE = MergeTree()
ORDER BY (line_id, detected_at);
```

## Control Chart Data - X-bar

```sql
SELECT
    station_id,
    dimension_id,
    toStartOfHour(measured_at)   AS hour,
    avg(measured_value)          AS xbar,
    stddevPop(measured_value)    AS sigma,
    avg(measured_value) + 3 * stddevPop(measured_value) AS ucl,
    avg(measured_value) - 3 * stddevPop(measured_value) AS lcl
FROM qc_measurements
WHERE measured_at >= today() - INTERVAL 7 DAY
GROUP BY station_id, dimension_id, hour
ORDER BY station_id, dimension_id, hour;
```

## Process Capability (Cpk)

```sql
WITH stats AS (
    SELECT
        dimension_id,
        avg(measured_value)     AS mean,
        stddevPop(measured_value) AS sigma,
        avg(nominal_value + tolerance_plus) AS usl,
        avg(nominal_value - tolerance_minus) AS lsl
    FROM qc_measurements
    WHERE station_id = 5
      AND measured_at >= today() - INTERVAL 30 DAY
    GROUP BY dimension_id
)
SELECT
    dimension_id,
    round((usl - mean) / (3 * sigma), 3) AS cpu,
    round((mean - lsl) / (3 * sigma), 3) AS cpl,
    round(least((usl - mean), (mean - lsl)) / (3 * sigma), 3) AS cpk
FROM stats;
```

## Defect Pareto Analysis

```sql
SELECT
    defect_code,
    any(defect_desc) AS description,
    count()          AS frequency,
    sum(count()) OVER (ORDER BY count() DESC) AS cumulative,
    sum(count()) OVER ()                      AS total,
    round(sum(count()) OVER (ORDER BY count() DESC) / sum(count()) OVER () * 100, 1) AS cum_pct
FROM defect_log
WHERE detected_at >= today() - INTERVAL 30 DAY
GROUP BY defect_code
ORDER BY frequency DESC;
```

## Out-of-Control Point Detection

```sql
WITH stats AS (
    SELECT
        station_id, dimension_id,
        avg(measured_value) AS mean,
        stddevPop(measured_value) AS sigma
    FROM qc_measurements
    WHERE measured_at BETWEEN today() - 30 AND today() - 1
    GROUP BY station_id, dimension_id
)
SELECT
    m.station_id,
    m.dimension_id,
    m.measured_at,
    m.measured_value,
    s.mean,
    abs(m.measured_value - s.mean) / s.sigma AS z_score
FROM qc_measurements m
JOIN stats s ON m.station_id = s.station_id AND m.dimension_id = s.dimension_id
WHERE m.measured_at >= today()
  AND abs(m.measured_value - s.mean) / s.sigma > 3
ORDER BY z_score DESC;
```

## Shift-Level Pass Rate

```sql
SELECT
    line_id,
    station_id,
    toStartOfInterval(measured_at, INTERVAL 8 HOUR) AS shift_start,
    count()                                         AS inspected,
    countIf(pass = 1)                               AS passed,
    round(countIf(pass = 1) / count() * 100, 2)     AS pass_rate_pct
FROM qc_measurements
WHERE measured_at >= today() - INTERVAL 30 DAY
GROUP BY line_id, station_id, shift_start
ORDER BY pass_rate_pct ASC
LIMIT 20;
```

## Summary

ClickHouse enables manufacturing quality teams to run SPC analytics at scale - control charts, capability indices, defect Pareto analysis, and out-of-control detection all run efficiently with standard SQL. The columnar engine compresses measurement data effectively, and fast aggregations support both real-time dashboards and month-end quality reports.
