# How to Use SimpleAggregateFunction Data Type in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Types, SimpleAggregateFunction, Aggregation

Description: Learn how SimpleAggregateFunction stores plain values that merge with simple operations like sum, min, max, and any in ClickHouse.

---

`SimpleAggregateFunction(func, T)` is a lightweight alternative to `AggregateFunction` for aggregation functions whose partial state is identical to the final result - such as `sum`, `min`, `max`, `any`, and a few others. Because no intermediate binary state is required, ClickHouse stores the column as a plain `T` value and simply applies the merge function when parts are combined. This makes `SimpleAggregateFunction` easy to work with: you insert ordinary values and read ordinary values, with no `-State`/`-Merge` combinator complexity.

## When to Use SimpleAggregateFunction

Use `SimpleAggregateFunction` for functions where `f(f(a, b), c) == f(a, b, c)` holds - that is, where the partial result can be merged by applying the same function again. Supported functions include `any`, `anyLast`, `min`, `max`, `sum`, `sumWithOverflow`, `groupBitAnd`, `groupBitOr`, `groupBitXor`, and `groupArrayArray`.

```sql
-- Supported aggregation functions for SimpleAggregateFunction
-- any, anyLast, min, max, sum, sumWithOverflow
-- groupBitAnd, groupBitOr, groupBitXor
-- groupArrayArray (array concatenation)
-- NOT supported: uniq, avg, quantile (use AggregateFunction for those)
SELECT 'see above list' AS reference;
```

## Using SimpleAggregateFunction with SummingMergeTree

`SummingMergeTree` sums numeric columns on merge, but `SimpleAggregateFunction` columns let you mix different merge strategies in the same table.

```sql
CREATE TABLE page_stats (
    date        Date,
    page        LowCardinality(String),
    views       SimpleAggregateFunction(sum, UInt64),
    unique_days SimpleAggregateFunction(max, UInt32),
    first_seen  SimpleAggregateFunction(min, DateTime),
    last_seen   SimpleAggregateFunction(max, DateTime)
) ENGINE = SummingMergeTree()
ORDER BY (date, page);

-- Insert raw event counts (each insert is a partial result)
INSERT INTO page_stats VALUES
    ('2026-03-01', '/home',    1500, 1, '2026-03-01 00:00:01', '2026-03-01 23:59:50'),
    ('2026-03-01', '/pricing', 320,  1, '2026-03-01 00:05:00', '2026-03-01 22:10:00');

INSERT INTO page_stats VALUES
    ('2026-03-01', '/home',    800,  1, '2026-03-01 01:00:00', '2026-03-01 23:59:59'),
    ('2026-03-01', '/pricing', 150,  1, '2026-03-01 08:00:00', '2026-03-01 21:00:00');

-- Query: totals are automatically merged on background merges
-- Use final or explicit aggregation to get correct results before merge
SELECT
    page,
    sum(views)     AS total_views,
    max(last_seen) AS latest_visit
FROM page_stats
GROUP BY page
ORDER BY total_views DESC;
```

## Using SimpleAggregateFunction with AggregatingMergeTree

`AggregatingMergeTree` is the standard engine for pre-aggregated tables. `SimpleAggregateFunction` columns work alongside `AggregateFunction` columns in the same table.

```sql
CREATE TABLE user_activity_agg (
    date          Date,
    user_segment  LowCardinality(String),
    total_events  SimpleAggregateFunction(sum,     UInt64),
    total_revenue SimpleAggregateFunction(sum,     Decimal(18, 4)),
    first_event   SimpleAggregateFunction(min,     DateTime),
    last_event    SimpleAggregateFunction(max,     DateTime),
    -- AggregateFunction for uniq (requires -State combinator)
    unique_users  AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree()
ORDER BY (date, user_segment);
```

## Inserting into SimpleAggregateFunction Columns

Insert plain values - no special combinator functions are needed.

```sql
INSERT INTO user_activity_agg
SELECT
    toDate(event_time)   AS date,
    segment              AS user_segment,
    count()              AS total_events,
    sum(revenue)         AS total_revenue,
    min(event_time)      AS first_event,
    max(event_time)      AS last_event,
    uniqState(user_id)   AS unique_users   -- AggregateFunction needs -State
FROM raw_events
GROUP BY date, user_segment;
```

## Querying SimpleAggregateFunction Columns

Read `SimpleAggregateFunction` columns directly without combiners. Use `FINAL` or explicit `GROUP BY` with the aggregate function to fold any unmerged duplicates.

```sql
-- Use FINAL to force merge of all parts before reading
SELECT
    date,
    user_segment,
    total_events,
    total_revenue,
    first_event,
    last_event
FROM user_activity_agg FINAL
ORDER BY date, user_segment;

-- Or use GROUP BY with the same function to merge manually
SELECT
    date,
    user_segment,
    sum(total_events)  AS events,
    sum(total_revenue) AS revenue,
    min(first_event)   AS first,
    max(last_event)    AS last
FROM user_activity_agg
GROUP BY date, user_segment
ORDER BY date, user_segment;
```

## Summary

`SimpleAggregateFunction(func, T)` is ideal when your aggregation function can merge partial states by re-applying itself - covering `sum`, `min`, `max`, and `any`. Unlike `AggregateFunction`, it stores plain values with no binary state overhead, making inserts and reads straightforward. Use it in `SummingMergeTree` or `AggregatingMergeTree` tables to pre-aggregate metrics efficiently, and combine it with `AggregateFunction` columns in the same table when you also need complex aggregates like `uniq` or `quantile`.
