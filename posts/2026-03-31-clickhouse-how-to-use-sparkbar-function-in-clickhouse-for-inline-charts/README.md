# How to Use sparkBar() Function in ClickHouse for Inline Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Visualization, SparkBar, Analytics, Terminal

Description: Learn how to use the sparkBar() function in ClickHouse to generate inline bar charts directly in query results for quick data visualization.

---

## Overview

`sparkBar()` is a ClickHouse function that renders a simple bar chart as a Unicode string directly in query output. It is useful for quickly visualizing distributions, histograms, and time-series in a terminal or log output without external tools.

## Basic Syntax

```sql
SELECT sparkBar(width, min, max)(value)
FROM table;
```

- `width`: number of characters in the output string
- `min`: minimum value of the range
- `max`: maximum value of the range
- `value`: the column to visualize

## Simple Histogram Example

Show a distribution of response times bucketed by 100ms:

```sql
SELECT
    intDiv(response_ms, 100) * 100 AS bucket,
    count()                        AS cnt,
    sparkBar(20, 0, max(count()) OVER ())(count()) AS bar
FROM api_requests
GROUP BY bucket
ORDER BY bucket;
```

```text
bucket | cnt  | bar
0      | 1523 | ████████████████████
100    | 892  | ████████████
200    | 445  | ██████
300    | 201  | ███
400    | 87   | █
```

## Daily Active Users Chart

```sql
SELECT
    event_date,
    uniq(user_id) AS dau,
    sparkBar(30, 0, max(uniq(user_id)) OVER ())(uniq(user_id)) AS chart
FROM page_views
WHERE event_date >= today() - 14
GROUP BY event_date
ORDER BY event_date;
```

## Per-Endpoint Request Rate

```sql
SELECT
    endpoint,
    count()                       AS requests,
    sparkBar(25, 0, max(count()) OVER ())(count()) AS bar
FROM http_access_log
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY endpoint
ORDER BY requests DESC
LIMIT 10;
```

## Visualizing Error Rate by Hour

```sql
SELECT
    toHour(event_time)                          AS hour,
    countIf(status_code >= 500) / count() * 100 AS error_pct,
    sparkBar(20, 0, 100)(
        countIf(status_code >= 500) / count() * 100
    ) AS error_bar
FROM http_access_log
WHERE event_date = today()
GROUP BY hour
ORDER BY hour;
```

## Multi-Column Spark Bars

```sql
SELECT
    toDate(ts) AS date,
    sparkBar(15, 0, 1000)(count())                              AS total_bar,
    sparkBar(15, 0, 100)(countIf(status >= 500))               AS error_bar,
    count()                                                     AS total,
    countIf(status >= 500)                                      AS errors
FROM requests
GROUP BY date
ORDER BY date DESC
LIMIT 7;
```

## Notes on Usage

- The output uses Unicode block characters: `█▇▆▅▄▃▂▁ `.
- Works well in ClickHouse CLI, `clickhouse-client`, and log pipelines.
- The `max() OVER ()` window function trick dynamically scales the bar relative to the largest value.
- It is a display function only and does not affect query performance significantly.

```sql
-- Fixed max value approach
SELECT
    day,
    sparkBar(20, 0, 10000)(event_count) AS bar,
    event_count
FROM daily_stats
ORDER BY day;
```

## Summary

`sparkBar()` lets you render inline Unicode bar charts directly inside ClickHouse query results. It is ideal for quick terminal dashboards, anomaly spotting, and data exploration without external visualization tools. Combine it with window functions to auto-scale bars relative to the dataset maximum.
