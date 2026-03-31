# How to Track Equipment OEE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, OEE, Manufacturing, Analytics, Production, Time Series

Description: Calculate Overall Equipment Effectiveness from raw machine state and production data in ClickHouse using shift-aware SQL queries.

---

## What Is OEE

Overall Equipment Effectiveness (OEE) is the gold-standard KPI for manufacturing performance. It combines three factors:

- Availability: percentage of planned time the machine was running
- Performance: actual throughput vs theoretical maximum speed
- Quality: good units as a percentage of total units produced

OEE = Availability x Performance x Quality

## Schema

```sql
CREATE TABLE machine_states
(
    machine_id   UInt32,
    line_id      UInt16,
    recorded_at  DateTime,
    state        UInt8,
    speed_pct    Float32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (machine_id, recorded_at)
TTL recorded_at + INTERVAL 3 YEAR;

CREATE TABLE production_counts
(
    machine_id    UInt32,
    recorded_at   DateTime,
    total_units   UInt32,
    good_units    UInt32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (machine_id, recorded_at);
```

States: 1 = running, 2 = idle, 3 = fault, 4 = changeover.

## Availability Calculation

```sql
SELECT
    machine_id,
    toDate(recorded_at)                                        AS day,
    countIf(state = 1)                                         AS running_min,
    countIf(state IN (1, 2, 3))                                AS planned_min,
    round(countIf(state = 1) / countIf(state IN (1, 2, 3)) * 100, 2) AS availability_pct
FROM machine_states
WHERE recorded_at >= today() - INTERVAL 30 DAY
GROUP BY machine_id, day
ORDER BY machine_id, day;
```

## Performance Calculation

```sql
SELECT
    s.machine_id,
    toDate(s.recorded_at)                                      AS day,
    sum(s.speed_pct * countIf(s.state = 1)) / countIf(s.state = 1) AS avg_speed_pct
FROM machine_states s
WHERE s.recorded_at >= today() - INTERVAL 30 DAY
GROUP BY s.machine_id, day;
```

## Quality Calculation

```sql
SELECT
    machine_id,
    toDate(recorded_at)                                           AS day,
    sum(good_units)                                               AS good,
    sum(total_units)                                              AS total,
    round(sum(good_units) / sum(total_units) * 100, 2)            AS quality_pct
FROM production_counts
WHERE recorded_at >= today() - INTERVAL 30 DAY
GROUP BY machine_id, day;
```

## Combined OEE Query

```sql
WITH avail AS (
    SELECT
        machine_id, toDate(recorded_at) AS day,
        round(countIf(state = 1) / countIf(state IN (1, 2, 3)) * 100, 2) AS a_pct
    FROM machine_states
    WHERE recorded_at >= today() - INTERVAL 30 DAY
    GROUP BY machine_id, day
),
perf AS (
    SELECT
        machine_id, toDate(recorded_at) AS day,
        round(avg(if(state = 1, speed_pct, NULL)), 2) AS p_pct
    FROM machine_states
    WHERE recorded_at >= today() - INTERVAL 30 DAY
    GROUP BY machine_id, day
),
qual AS (
    SELECT
        machine_id, toDate(recorded_at) AS day,
        round(sum(good_units) / sum(total_units) * 100, 2) AS q_pct
    FROM production_counts
    WHERE recorded_at >= today() - INTERVAL 30 DAY
    GROUP BY machine_id, day
)
SELECT
    a.machine_id,
    a.day,
    a.a_pct AS availability,
    p.p_pct AS performance,
    q.q_pct AS quality,
    round(a.a_pct * p.p_pct * q.q_pct / 10000, 2) AS oee_pct
FROM avail a
JOIN perf p  ON a.machine_id = p.machine_id AND a.day = p.day
JOIN qual q  ON a.machine_id = q.machine_id AND a.day = q.day
ORDER BY a.machine_id, a.day;
```

## OEE Materialized View

```sql
CREATE TABLE machine_oee_daily
(
    machine_id     UInt32,
    day            Date,
    availability   Float32,
    performance    Float32,
    quality        Float32,
    oee            Float32
)
ENGINE = ReplacingMergeTree()
ORDER BY (machine_id, day);
```

Populate this view from a scheduled INSERT ... SELECT using the combined query above to keep dashboard queries fast.

## Summary

Tracking OEE in ClickHouse requires storing machine state and production count data at the event level, then using SQL aggregations to compute availability, performance, and quality per shift or day. With materialized or pre-computed OEE tables, dashboards can display results instantly for hundreds of machines.
