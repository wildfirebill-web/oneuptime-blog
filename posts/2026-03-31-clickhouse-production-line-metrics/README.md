# How to Track Production Line Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Production, Manufacturing, Analytics, Throughput, OEE

Description: Track production line performance metrics in ClickHouse including throughput, takt time compliance, bottleneck identification, and shift-by-shift comparisons.

---

Production line analytics requires tracking parts output, cycle times, and bottleneck machines in real time. ClickHouse aggregates high-frequency production signals to surface actionable metrics for line supervisors and engineers.

## Production Records Table

```sql
CREATE TABLE production_records (
    record_id    UUID,
    plant_id     LowCardinality(String),
    line_id      LowCardinality(String),
    workstation  LowCardinality(String),
    part_number  LowCardinality(String),
    quantity     UInt32,
    good_qty     UInt32,
    scrap_qty    UInt32,
    cycle_time_s Float32,
    operator_id  LowCardinality(String),
    shift        LowCardinality(String),
    produced_at  DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(produced_at)
ORDER BY (plant_id, line_id, produced_at);
```

## Hourly Throughput by Line

```sql
SELECT
    line_id,
    toStartOfHour(produced_at) AS hour,
    sum(quantity) AS total_produced,
    sum(good_qty) AS good_parts,
    sum(scrap_qty) AS scrap_parts,
    round(sum(good_qty) / nullIf(sum(quantity), 0) * 100, 2) AS yield_pct
FROM production_records
WHERE produced_at >= today()
GROUP BY line_id, hour
ORDER BY line_id, hour;
```

## Bottleneck Workstation Identification

Find workstations with the highest average cycle time - potential bottlenecks:

```sql
SELECT
    line_id,
    workstation,
    avg(cycle_time_s) AS avg_cycle_s,
    max(cycle_time_s) AS max_cycle_s,
    sum(quantity) AS parts_produced,
    round(avg(cycle_time_s), 2) AS bottleneck_score
FROM production_records
WHERE produced_at >= today() - 7
  AND cycle_time_s > 0
GROUP BY line_id, workstation
ORDER BY line_id, avg_cycle_s DESC;
```

## Shift Performance Comparison

Compare shift output and quality side by side:

```sql
SELECT
    line_id,
    shift,
    toDate(produced_at) AS day,
    sum(quantity) AS total_parts,
    sum(good_qty) AS good_parts,
    round(sum(good_qty) / nullIf(sum(quantity), 0) * 100, 2) AS first_pass_yield_pct,
    avg(cycle_time_s) AS avg_cycle_s
FROM production_records
WHERE produced_at >= today() - 14
GROUP BY line_id, shift, day
ORDER BY day, line_id, shift;
```

## Part Number Performance

Identify which part numbers have highest scrap rates:

```sql
SELECT
    part_number,
    sum(quantity) AS total_produced,
    sum(scrap_qty) AS total_scrap,
    round(sum(scrap_qty) / nullIf(sum(quantity), 0) * 100, 2) AS scrap_rate_pct,
    avg(cycle_time_s) AS avg_cycle_s
FROM production_records
WHERE produced_at >= today() - 30
GROUP BY part_number
HAVING scrap_rate_pct > 2
ORDER BY scrap_rate_pct DESC;
```

## Operator Productivity

Benchmark operator performance by parts per hour:

```sql
SELECT
    operator_id,
    shift,
    count() AS records,
    sum(quantity) AS total_parts,
    sum(good_qty) AS good_parts,
    round(sum(quantity) / nullIf(count() * avg(cycle_time_s) / 3600, 0), 2) AS parts_per_hour
FROM production_records
WHERE produced_at >= today() - 7
GROUP BY operator_id, shift
ORDER BY parts_per_hour DESC;
```

## Summary

ClickHouse enables comprehensive production line analytics - from real-time throughput monitoring to bottleneck identification, shift comparisons, and operator benchmarking. Fast columnar aggregations over production events make it practical to deliver live metrics to shop floor dashboards.
