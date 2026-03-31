# How to Use sparkBar() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, SparkBar, Visualization

Description: Render inline ASCII bar charts inside ClickHouse query results using the sparkBar() aggregate function with configurable width and value range.

---

ClickHouse's `sparkBar()` function lets you produce a compact, ASCII-style bar chart directly inside a result set - no external tool required. Each character in the returned string represents a relative magnitude, making it possible to eyeball trends and distributions without leaving your SQL client.

## Syntax

```sql
sparkBar(width, min, max)(value, height)
```

- `width` - number of characters (buckets) in the output string.
- `min` / `max` - the expected value range; values outside this range are clamped.
- `value` - the x-axis position (often a timestamp or sequential integer).
- `height` - the y-axis magnitude to visualize.

The function is an aggregate: it groups incoming `(value, height)` pairs into `width` buckets and fills each bucket with a Unicode block character proportional to the maximum height seen in that bucket.

## Basic Example

```sql
SELECT sparkBar(10, 1, 10)(number + 1, number + 1) AS bar
FROM numbers(10);
```

```text
bar
----------
▁▂▃▄▅▆▇█▉█
```

## Visualizing Time-Series Data

Create a table of hourly request counts and render a 24-character sparkline per day:

```sql
CREATE TABLE requests
(
    ts        DateTime,
    status    UInt16,
    duration  UInt32
)
ENGINE = MergeTree()
ORDER BY ts;

INSERT INTO requests
SELECT
    toDateTime('2024-01-15 00:00:00') + (number * 3600) AS ts,
    200                                                  AS status,
    50 + (rand() % 200)                                  AS duration
FROM numbers(24);
```

```sql
SELECT
    toDate(ts)                                    AS day,
    sparkBar(24, 0, 500)(toHour(ts), duration)    AS latency_sparkline
FROM requests
GROUP BY day
ORDER BY day;
```

```text
day        | latency_sparkline
-----------|-------------------------
2024-01-15 | ▃▅▇▂▆▁▄▆█▃▅▂▇▄▁▆▃▅▂▄▇▁▅▃
```

Each character position corresponds to one hour of the day, and taller bars represent higher latency.

## Controlling Width and Range

Choosing good `min`/`max` values is important. Values below `min` map to an empty bucket; values above `max` are clamped to the tallest bar:

```sql
-- Narrow range highlights micro-differences
SELECT sparkBar(20, 100, 200)(toHour(ts), duration) AS bar
FROM requests
GROUP BY toDate(ts);

-- Wide range with 5 characters - quick overview
SELECT sparkBar(5, 0, 1000)(toHour(ts), duration) AS bar
FROM requests
GROUP BY toDate(ts);
```

## Comparing Multiple Metrics Side-by-Side

```sql
SELECT
    toDate(ts)                                        AS day,
    sparkBar(12, 0, 500)(toHour(ts), duration)        AS latency,
    sparkBar(12, 0, 100)(toHour(ts), status = 200)    AS success_rate
FROM requests
GROUP BY day
ORDER BY day;
```

This places a latency sparkline and a success-rate sparkline in adjacent columns, enabling a quick visual correlation check.

## Combining sparkBar() with Other Aggregates

```sql
SELECT
    toDate(ts)                                      AS day,
    count()                                         AS total_requests,
    round(avg(duration), 1)                         AS avg_ms,
    sparkBar(24, 0, 500)(toHour(ts), duration)      AS hourly_latency
FROM requests
GROUP BY day
ORDER BY day;
```

## Summary

`sparkBar(width, min, max)(value, height)` renders an inline ASCII bar chart directly in query output, making time-series and distribution data immediately readable in any SQL terminal. Choose `width` to match the number of x-axis buckets you need and set `min`/`max` to the expected value range to get the best contrast. Pair it with standard aggregates like `avg()` or `count()` in the same `SELECT` for quick at-a-glance dashboards.
