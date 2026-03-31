# How to Use groupArraySample() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, groupArraySample, Sampling

Description: Learn how to randomly sample a fixed number of values from a group in ClickHouse using groupArraySample() with optional seed control.

---

`groupArraySample()` is a ClickHouse aggregate function that collects a random sample of up to `N` values from each group and returns them as an array. Unlike `groupArray()`, which collects every value, `groupArraySample()` lets you cap the result size without writing separate LIMIT logic per group. This is particularly useful for preview data, debugging large aggregations, or feeding downstream sampling pipelines without full scans.

## Syntax

```sql
-- Sample up to N values, random seed per execution
groupArraySample(N)(column)

-- Sample up to N values with a fixed seed for reproducibility
groupArraySample(N, seed)(column)
```

`N` is the maximum number of elements to include in the output array. If the group contains fewer than `N` rows, all values are returned. The `seed` parameter makes the selection deterministic - the same query with the same data returns the same sample every time.

## Basic Example

```sql
SELECT
    category,
    groupArraySample(5)(product_name) AS sample_products
FROM products
GROUP BY category;
```

Each category row returns an array of up to five randomly chosen product names. The sample is drawn using reservoir sampling, which guarantees uniform probability for all group members.

## Reproducible Sampling with a Seed

```sql
SELECT
    region,
    groupArraySample(10, 42)(user_id) AS sampled_users
FROM user_events
WHERE event_date = today() - 1
GROUP BY region;
```

Using seed `42` ensures you get the same ten user IDs for each region every time you run the query against unchanged data. This is useful for A/B test verification or reproducible report snapshots.

## Sampling for Data Preview

When exploring a large table, `groupArraySample` gives you representative example values per group without a correlated subquery.

```sql
-- Preview up to 3 raw log messages per log level
SELECT
    level,
    groupArraySample(3)(message) AS example_messages
FROM application_logs
WHERE log_date >= today() - 7
GROUP BY level;
```

## Combining with Other Aggregates

`groupArraySample` can sit alongside standard aggregates in the same SELECT.

```sql
SELECT
    event_type,
    count()                              AS total_events,
    avg(value)                           AS avg_value,
    groupArraySample(5, 123)(session_id) AS sample_sessions
FROM events
WHERE created_at >= now() - INTERVAL 1 DAY
GROUP BY event_type
ORDER BY total_events DESC;
```

## Unnesting the Sample Array

Use `ARRAY JOIN` to expand sampled values into individual rows for further processing.

```sql
SELECT
    category,
    sampled_user
FROM (
    SELECT
        category,
        groupArraySample(10)(user_id) AS sampled_users
    FROM purchases
    GROUP BY category
)
ARRAY JOIN sampled_users AS sampled_user;
```

## Comparing groupArraySample vs groupArray

```sql
-- groupArray collects ALL values - can be huge
SELECT category, length(groupArray(product_id)) AS total
FROM products
GROUP BY category;

-- groupArraySample caps the result - safe memory usage
SELECT category, length(groupArraySample(100)(product_id)) AS sampled
FROM products
GROUP BY category;
```

For groups with millions of rows, `groupArray` may exhaust memory limits. `groupArraySample` keeps memory bounded at `N` elements per group.

## Checking Sample Distribution

You can verify that the sample is spread evenly by comparing the sample size to the full group count.

```sql
SELECT
    region,
    count()                              AS total_rows,
    length(groupArraySample(50)(user_id)) AS sample_size
FROM user_events
GROUP BY region;
```

When total_rows is less than 50, sample_size equals total_rows, confirming the function returns all available values rather than padding with duplicates.

## Summary

`groupArraySample(N)(col)` is the right tool when you need a bounded random subset of values from each group in ClickHouse. Providing an optional seed makes sampling reproducible, which is essential for testing and reporting workflows. Because the function uses reservoir sampling, the probability of each row being selected is uniform, and memory usage stays proportional to `N` regardless of group size.
