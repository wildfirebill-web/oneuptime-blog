# How to Analyze Manufacturing Yield Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Manufacturing Yield, Rolled Throughput Yield, Quality, Analytics

Description: Analyze manufacturing yield data in ClickHouse to compute first pass yield, rolled throughput yield, and identify the steps with the highest yield loss.

---

Manufacturing yield analysis determines how efficiently a process converts raw materials into good product. ClickHouse handles large inspection datasets to compute First Pass Yield, Rolled Throughput Yield (RTY), and step-level yield loss.

## Yield Records Table

```sql
CREATE TABLE yield_records (
    record_id     UUID,
    plant_id      LowCardinality(String),
    line_id       LowCardinality(String),
    process_step  LowCardinality(String),
    step_sequence UInt8,
    part_number   LowCardinality(String),
    batch_id      String,
    input_qty     UInt32,
    output_qty    UInt32,
    scrap_qty     UInt32,
    rework_qty    UInt32,
    shift         LowCardinality(String),
    recorded_at   DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (plant_id, line_id, process_step, recorded_at);
```

## First Pass Yield by Step

```sql
SELECT
    line_id,
    process_step,
    step_sequence,
    sum(input_qty) AS total_input,
    sum(output_qty) AS total_output,
    sum(scrap_qty) AS total_scrap,
    sum(rework_qty) AS total_rework,
    round(sum(output_qty) / nullIf(sum(input_qty), 0) * 100, 2) AS fpy_pct
FROM yield_records
WHERE recorded_at >= today() - 30
GROUP BY line_id, process_step, step_sequence
ORDER BY line_id, step_sequence;
```

## Rolled Throughput Yield (RTY)

RTY = product of all step FPYs - measures probability a unit goes through all steps defect-free:

```sql
SELECT
    line_id,
    exp(sum(log(fpy_decimal))) AS rty,
    round(exp(sum(log(fpy_decimal))) * 100, 2) AS rty_pct
FROM (
    SELECT
        line_id,
        process_step,
        sum(output_qty) / nullIf(sum(input_qty), 0) AS fpy_decimal
    FROM yield_records
    WHERE recorded_at >= today() - 30
    GROUP BY line_id, process_step
    HAVING fpy_decimal > 0
)
GROUP BY line_id
ORDER BY rty_pct;
```

## Yield Loss by Step (Waterfall)

Identify the biggest yield loss steps:

```sql
SELECT
    process_step,
    step_sequence,
    sum(scrap_qty) AS scrap_units,
    sum(rework_qty) AS rework_units,
    sum(input_qty) AS total_input,
    round(sum(scrap_qty) / nullIf(sum(input_qty), 0) * 100, 2) AS scrap_rate_pct,
    round(sum(rework_qty) / nullIf(sum(input_qty), 0) * 100, 2) AS rework_rate_pct
FROM yield_records
WHERE recorded_at >= today() - 30
GROUP BY process_step, step_sequence
ORDER BY scrap_rate_pct DESC;
```

## Yield Trend Over Time

```sql
SELECT
    line_id,
    toStartOfWeek(recorded_at) AS week,
    round(sum(output_qty) / nullIf(sum(input_qty), 0) * 100, 2) AS weekly_fpy_pct,
    weekly_fpy_pct - lag(weekly_fpy_pct, 1) OVER (PARTITION BY line_id ORDER BY week) AS fpy_change
FROM yield_records
WHERE recorded_at >= today() - 90
GROUP BY line_id, week
ORDER BY line_id, week;
```

## Yield by Shift

Compare yield performance across shifts to identify training needs:

```sql
SELECT
    line_id,
    shift,
    toDate(recorded_at) AS day,
    round(sum(output_qty) / nullIf(sum(input_qty), 0) * 100, 2) AS fpy_pct,
    sum(scrap_qty) AS scrap_units,
    sum(rework_qty) AS rework_units
FROM yield_records
WHERE recorded_at >= today() - 14
GROUP BY line_id, shift, day
ORDER BY day, line_id, shift;
```

## Summary

ClickHouse makes manufacturing yield analysis fast and scalable - First Pass Yield, Rolled Throughput Yield, step-level loss analysis, and trend tracking are all expressible in SQL. These metrics drive Six Sigma improvement projects and help prioritize process step investments for maximum yield gain.
