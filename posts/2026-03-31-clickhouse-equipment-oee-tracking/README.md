# How to Track Equipment OEE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, OEE, Manufacturing, Availability, Performance, Quality

Description: Track Overall Equipment Effectiveness (OEE) in ClickHouse by computing availability, performance, and quality scores for manufacturing equipment.

---

Overall Equipment Effectiveness (OEE) is the gold standard KPI for manufacturing productivity. It combines Availability, Performance, and Quality into a single score. ClickHouse can compute OEE metrics from machine event data efficiently.

## OEE Calculation Inputs Table

```sql
CREATE TABLE oee_events (
    machine_id    LowCardinality(String),
    line_id       LowCardinality(String),
    plant_id      LowCardinality(String),
    state         LowCardinality(String),  -- running, planned_stop, unplanned_stop, changeover
    parts_produced UInt32,
    good_parts    UInt32,
    ideal_rate_pph UInt16,  -- ideal parts per hour
    event_at      DateTime,
    duration_min  UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(event_at)
ORDER BY (plant_id, line_id, machine_id, event_at);
```

## OEE Components Per Shift

```sql
SELECT
    machine_id,
    line_id,
    toDate(event_at) AS day,
    multiIf(toHour(event_at) < 8, 'Night', toHour(event_at) < 16, 'Day', 'Evening') AS shift,

    -- Availability = running time / planned production time
    sumIf(duration_min, state = 'running') AS run_time_min,
    sumIf(duration_min, state NOT IN ('planned_stop')) AS planned_time_min,
    round(sumIf(duration_min, state = 'running') / nullIf(sumIf(duration_min, state NOT IN ('planned_stop')), 0), 4) AS availability,

    -- Performance = actual output / theoretical max output
    sum(parts_produced) AS actual_output,
    sumIf(duration_min, state = 'running') * max(ideal_rate_pph) / 60 AS ideal_output,
    round(sum(parts_produced) / nullIf(sumIf(duration_min, state = 'running') * max(ideal_rate_pph) / 60, 0), 4) AS performance,

    -- Quality = good parts / total parts
    sum(good_parts) AS good_output,
    round(sum(good_parts) / nullIf(sum(parts_produced), 0), 4) AS quality,

    -- OEE = Availability * Performance * Quality
    round(
        (sumIf(duration_min, state = 'running') / nullIf(sumIf(duration_min, state NOT IN ('planned_stop')), 0)) *
        (sum(parts_produced) / nullIf(sumIf(duration_min, state = 'running') * max(ideal_rate_pph) / 60, 0)) *
        (sum(good_parts) / nullIf(sum(parts_produced), 0)) * 100, 2
    ) AS oee_pct

FROM oee_events
WHERE event_at >= today() - 7
GROUP BY machine_id, line_id, day, shift
ORDER BY day, line_id, machine_id;
```

## OEE Trend Over Time

Track weekly OEE trend per line:

```sql
SELECT
    line_id,
    toStartOfWeek(event_at) AS week,
    round(
        avg(availability) * avg(performance) * avg(quality) * 100, 2
    ) AS weekly_oee_pct
FROM (
    SELECT
        line_id,
        toDate(event_at) AS day,
        sumIf(duration_min, state = 'running') / nullIf(sumIf(duration_min, state != 'planned_stop'), 0) AS availability,
        sum(parts_produced) / nullIf(sumIf(duration_min, state = 'running') * max(ideal_rate_pph) / 60, 0) AS performance,
        sum(good_parts) / nullIf(sum(parts_produced), 0) AS quality
    FROM oee_events
    WHERE event_at >= today() - 90
    GROUP BY line_id, day
)
GROUP BY line_id, week
ORDER BY line_id, week;
```

## Availability Loss Breakdown

Identify the biggest sources of availability loss:

```sql
SELECT
    machine_id,
    state,
    sum(duration_min) AS total_loss_min,
    round(sum(duration_min) / sum(sum(duration_min)) OVER (PARTITION BY machine_id) * 100, 2) AS pct_of_losses
FROM oee_events
WHERE state IN ('unplanned_stop', 'changeover')
  AND event_at >= today() - 30
GROUP BY machine_id, state
ORDER BY machine_id, total_loss_min DESC;
```

## World-Class OEE Benchmarking

Flag machines below world-class OEE threshold of 85%:

```sql
SELECT
    machine_id,
    line_id,
    round(
        (sumIf(duration_min, state = 'running') / nullIf(sumIf(duration_min, state != 'planned_stop'), 0)) *
        (sum(parts_produced) / nullIf(sumIf(duration_min, state = 'running') * max(ideal_rate_pph) / 60, 0)) *
        (sum(good_parts) / nullIf(sum(parts_produced), 0)) * 100, 2
    ) AS oee_pct,
    oee_pct < 85 AS below_world_class
FROM oee_events
WHERE event_at >= today() - 30
GROUP BY machine_id, line_id
ORDER BY oee_pct;
```

## Summary

ClickHouse is well-suited for OEE tracking - it aggregates availability, performance, and quality components from machine event streams and computes composite OEE scores per machine, line, and shift. Time-series partitioning and fast SQL aggregations enable real-time OEE dashboards across entire plants.
