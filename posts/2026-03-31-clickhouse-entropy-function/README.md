# How to Use entropy() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Entropy, Information Theory

Description: Learn how to use the entropy() aggregate function in ClickHouse to calculate Shannon entropy and measure data diversity or detect anomalies.

---

Shannon entropy is a measure from information theory that quantifies the unpredictability or diversity of a dataset. ClickHouse's `entropy()` aggregate function computes Shannon entropy over a column's values, making it easy to detect anomalies, measure data diversity, and analyze distributions directly in SQL.

## What Is Shannon Entropy

Shannon entropy is defined as the sum of `-p * log2(p)` for each unique value, where `p` is the probability of that value occurring in the dataset. A result of `0` means all values are identical, while higher values indicate greater diversity.

- **0** - all values are the same (no information)
- **Higher values** - more diverse distribution (more information)
- **log2(n)** - maximum entropy when all n unique values are equally likely

## Syntax

```sql
entropy(col)
```

The function accepts a single column and returns a `Float64` representing the Shannon entropy in bits.

## Basic Example

Calculate the entropy of a categorical column to understand value distribution:

```sql
SELECT entropy(status)
FROM orders;
```

```sql
-- Sample result
-- entropy(status) = 1.58 (3 roughly equal categories: pending, shipped, delivered)
```

## Creating Sample Data

```sql
CREATE TABLE user_events
(
    user_id  UInt32,
    event    String,
    ts       DateTime
)
ENGINE = MergeTree()
ORDER BY ts;

INSERT INTO user_events VALUES
    (1, 'click',    '2024-01-01 10:00:00'),
    (1, 'purchase', '2024-01-01 10:05:00'),
    (2, 'click',    '2024-01-01 10:10:00'),
    (2, 'click',    '2024-01-01 10:15:00'),
    (3, 'view',     '2024-01-01 10:20:00'),
    (3, 'click',    '2024-01-01 10:25:00'),
    (4, 'view',     '2024-01-01 10:30:00'),
    (4, 'view',     '2024-01-01 10:35:00');
```

## Measuring Data Diversity

Use `entropy()` to compare how evenly events are distributed per user:

```sql
SELECT
    user_id,
    entropy(event) AS event_diversity,
    count()        AS total_events
FROM user_events
GROUP BY user_id
ORDER BY event_diversity DESC;
```

A user with `event_diversity = 0` performs only one type of action, while higher values indicate varied behavior.

## Anomaly Detection with Entropy

Low entropy in a time window can signal anomalous or bot-like behavior (e.g., a single repeated action):

```sql
SELECT
    toStartOfHour(ts)   AS hour,
    entropy(event)      AS hour_entropy,
    count()             AS event_count
FROM user_events
GROUP BY hour
ORDER BY hour;
```

Flag hours where entropy drops below a threshold:

```sql
SELECT
    toStartOfHour(ts) AS hour,
    entropy(event)    AS hour_entropy
FROM user_events
GROUP BY hour
HAVING hour_entropy < 0.5
ORDER BY hour;
```

## Comparing Entropy Across Segments

Compare the diversity of event types between two user segments:

```sql
SELECT
    multiIf(user_id <= 2, 'segment_a', 'segment_b') AS segment,
    entropy(event)                                   AS event_entropy
FROM user_events
GROUP BY segment;
```

## Entropy on Numeric Columns

`entropy()` works on any data type by treating each distinct value as a category:

```sql
CREATE TABLE sensor_readings
(
    sensor_id UInt32,
    reading   UInt8
)
ENGINE = MergeTree()
ORDER BY sensor_id;

INSERT INTO sensor_readings VALUES
    (1, 5), (1, 5), (1, 5), (1, 5),
    (2, 1), (2, 2), (2, 3), (2, 4);

SELECT
    sensor_id,
    entropy(reading) AS reading_entropy
FROM sensor_readings
GROUP BY sensor_id;
-- sensor 1: entropy = 0 (all same value)
-- sensor 2: entropy = 2.0 (four equally likely values)
```

## Combining Entropy with Other Aggregates

```sql
SELECT
    event,
    entropy(user_id)       AS user_id_entropy,
    uniq(user_id)          AS unique_users,
    count()                AS occurrences
FROM user_events
GROUP BY event
ORDER BY occurrences DESC;
```

## Summary

The `entropy()` function in ClickHouse brings Shannon entropy calculations directly into SQL, enabling data diversity measurement, anomaly detection, and behavioral analysis without external tooling. It operates on any column type, returns a `Float64` value in bits, and pairs naturally with `GROUP BY` for per-segment analysis. Use it to flag low-diversity time windows, compare user segments, or validate that categorical distributions are as expected.
