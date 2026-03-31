# How to Merge Overlapping Intervals in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Interval Merge, Overlap, Window Function, Analytics

Description: Learn how to merge overlapping time intervals in ClickHouse using window functions to calculate contiguous coverage from event start and end times.

---

## The Overlapping Intervals Problem

Overlapping intervals appear in calendar scheduling, session coverage, SLA downtime calculations, and resource utilization. Merging overlaps gives the total unique covered duration.

## Sample Intervals Table

```sql
CREATE TABLE sessions
(
    session_id UInt32,
    user_id UInt32,
    start_time DateTime,
    end_time DateTime
)
ENGINE = MergeTree()
ORDER BY (user_id, start_time);

INSERT INTO sessions VALUES
(1, 100, '2024-01-01 10:00:00', '2024-01-01 10:30:00'),
(2, 100, '2024-01-01 10:15:00', '2024-01-01 10:45:00'),
(3, 100, '2024-01-01 11:00:00', '2024-01-01 11:30:00'),
(4, 100, '2024-01-01 11:25:00', '2024-01-01 12:00:00'),
(5, 200, '2024-01-01 09:00:00', '2024-01-01 09:20:00');
```

Sessions 1 and 2 overlap (10:00-10:30 and 10:15-10:45), so they should merge to 10:00-10:45.

## Step 1 - Identify New Group Starts

A new merged group starts when an interval begins after the maximum end time seen so far:

```sql
SELECT
    user_id,
    start_time,
    end_time,
    max(end_time) OVER (
        PARTITION BY user_id
        ORDER BY start_time
        ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
    ) AS max_prev_end
FROM sessions
ORDER BY user_id, start_time;
```

## Step 2 - Assign Group IDs

```sql
WITH gaps AS (
    SELECT
        user_id,
        start_time,
        end_time,
        if(
            start_time > max(end_time) OVER (
                PARTITION BY user_id ORDER BY start_time
                ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
            ) OR row_number() OVER (PARTITION BY user_id ORDER BY start_time) = 1,
            1, 0
        ) AS is_new_group
    FROM sessions
)
SELECT
    user_id,
    start_time,
    end_time,
    sum(is_new_group) OVER (PARTITION BY user_id ORDER BY start_time) AS group_id
FROM gaps;
```

## Step 3 - Merge Each Group

```sql
WITH gaps AS (
    SELECT
        user_id, start_time, end_time,
        if(
            start_time > max(end_time) OVER (
                PARTITION BY user_id ORDER BY start_time
                ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
            ) OR row_number() OVER (PARTITION BY user_id ORDER BY start_time) = 1,
            1, 0
        ) AS is_new_group
    FROM sessions
),
grouped AS (
    SELECT user_id, start_time, end_time,
        sum(is_new_group) OVER (PARTITION BY user_id ORDER BY start_time) AS group_id
    FROM gaps
)
SELECT
    user_id,
    group_id,
    min(start_time) AS merged_start,
    max(end_time) AS merged_end,
    dateDiff('minute', min(start_time), max(end_time)) AS duration_minutes
FROM grouped
GROUP BY user_id, group_id
ORDER BY user_id, merged_start;
```

## Calculating Total Coverage Time

```sql
-- Total non-overlapping covered minutes per user
SELECT
    user_id,
    sum(dateDiff('minute', merged_start, merged_end)) AS total_covered_minutes
FROM (
    -- previous merged intervals query here
)
GROUP BY user_id;
```

## Summary

Merging overlapping intervals in ClickHouse uses a three-step window function approach: identify new group starts where the interval begins after the previous maximum end time, assign group IDs with cumulative sum, then aggregate `min(start)` and `max(end)` per group. This computes accurate non-overlapping coverage for scheduling and SLA calculations.
