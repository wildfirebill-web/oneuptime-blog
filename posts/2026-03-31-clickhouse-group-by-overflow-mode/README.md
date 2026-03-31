# How to Set group_by_overflow_mode in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Database, Performance, Configuration, Query Optimization

Description: Learn how to configure group_by_overflow_mode in ClickHouse to control query behavior when GROUP BY aggregation exceeds memory or row count limits, and avoid OOM errors.

---

When a `GROUP BY` query in ClickHouse accumulates too many unique group keys, it can exhaust the server's RAM. ClickHouse lets you set upper bounds on GROUP BY state size via `max_rows_to_group_by` and `max_bytes_before_external_group_by`. The `group_by_overflow_mode` setting controls what happens when those limits are exceeded.

## The Problem: Unbounded GROUP BY

A query that groups by a high-cardinality column against a large table can create millions of groups, each holding partial aggregation state:

```sql
-- This could accumulate hundreds of millions of unique session_ids
SELECT
    session_id,
    count() AS events,
    sum(duration_ms) AS total_duration
FROM web_events
GROUP BY session_id;
```

If the server only has 16 GiB available for this query and the aggregation state grows beyond that, the query fails with an out-of-memory error unless you have configured a response policy.

## The Three Overflow Modes

`group_by_overflow_mode` accepts three values:

| Mode | Behavior |
|---|---|
| `throw` | Raise an exception and cancel the query (default) |
| `break` | Stop adding new groups; aggregate only the groups already seen |
| `any` | Same as `break`, but return approximate results for remaining rows |

## throw Mode (Default)

```sql
-- Query fails if more than 1 million unique groups are created
SELECT
    user_id,
    count() AS events
FROM events
GROUP BY user_id
SETTINGS
    max_rows_to_group_by = 1000000,
    group_by_overflow_mode = 'throw';
```

Use `throw` when correctness is non-negotiable and you want failed queries to be visible as errors rather than silently returning partial results.

## break Mode

```sql
-- Stop adding new groups after 5 million; aggregate remaining rows into existing groups
SELECT
    user_country,
    device_type,
    count() AS sessions
FROM sessions
GROUP BY user_country, device_type
SETTINGS
    max_rows_to_group_by = 5000000,
    group_by_overflow_mode = 'break';
```

With `break`, ClickHouse stops creating new group keys once the limit is reached. Rows whose keys were already seen continue to accumulate into their existing groups. Rows with entirely new keys are dropped.

The result is **partially correct**: groups that appeared early in the scan are complete, groups that only appeared after the limit are missing entirely.

## any Mode

```sql
-- Approximate: remaining rows assigned to any existing group (cheapest fallback)
SELECT
    product_category,
    count() AS orders,
    sum(amount) AS revenue
FROM orders
GROUP BY product_category
SETTINGS
    max_rows_to_group_by = 2000000,
    group_by_overflow_mode = 'any';
```

`any` behaves like `break` but is intended for cases where you explicitly want approximate results and accept that some counts may be inflated by rows that should have gone into missing groups.

## Setting Limits That Trigger Overflow Mode

`group_by_overflow_mode` only takes effect when a limit is also set:

```sql
-- max_rows_to_group_by: limit on number of unique group keys
-- max_bytes_before_external_group_by: spill to disk when aggregation state exceeds this size

SELECT
    user_id,
    toYYYYMM(event_date) AS month,
    count() AS events
FROM events
GROUP BY user_id, month
SETTINGS
    max_rows_to_group_by = 10000000,          -- 10 million groups
    group_by_overflow_mode = 'break',
    max_bytes_before_external_group_by = 8589934592; -- 8 GiB disk spill threshold
```

The two limits are complementary:
- `max_rows_to_group_by` caps the number of distinct keys.
- `max_bytes_before_external_group_by` caps memory usage by spilling to disk before triggering overflow.

Setting `max_bytes_before_external_group_by` is often preferable to relying on `group_by_overflow_mode` because disk spill preserves correctness while overflow modes introduce approximation or failure.

## Configuring Defaults in User Profiles

```xml
<clickhouse>
    <profiles>

        <!-- For dashboard users: prefer break over throw to avoid error pages -->
        <dashboard>
            <max_rows_to_group_by>5000000</max_rows_to_group_by>
            <group_by_overflow_mode>break</group_by_overflow_mode>
            <max_bytes_before_external_group_by>4294967296</max_bytes_before_external_group_by>
        </dashboard>

        <!-- For engineers: throw so they know when queries exceed bounds -->
        <engineer>
            <max_rows_to_group_by>50000000</max_rows_to_group_by>
            <group_by_overflow_mode>throw</group_by_overflow_mode>
            <max_bytes_before_external_group_by>10737418240</max_bytes_before_external_group_by>
        </engineer>

        <!-- For approximate analytics pipelines: any is acceptable -->
        <analytics_approx>
            <max_rows_to_group_by>10000000</max_rows_to_group_by>
            <group_by_overflow_mode>any</group_by_overflow_mode>
        </analytics_approx>

    </profiles>
</clickhouse>
```

## Combining with totals_mode

When using `WITH TOTALS`, the `totals_mode` setting interacts with `group_by_overflow_mode`:

```sql
-- WITH TOTALS: how are rows after the overflow handled for totals calculation?
SELECT
    user_country,
    count() AS users
FROM users
GROUP BY user_country
WITH TOTALS
SETTINGS
    max_rows_to_group_by = 1000,
    group_by_overflow_mode = 'any',
    totals_mode = 'after_having_inclusive';
    -- Options: after_having_inclusive, after_having_exclusive,
    --          before_having, after_having_auto
```

## Detecting Overflow at Runtime

```sql
-- Check query log for queries that hit row limits
SELECT
    query_id,
    user,
    query_duration_ms,
    ProfileEvents['GroupByOverflowModeBreak'] AS overflow_breaks,
    left(query, 100) AS query_snippet
FROM system.query_log
WHERE type = 'QueryFinish'
  AND ProfileEvents['GroupByOverflowModeBreak'] > 0
  AND event_date = today()
ORDER BY event_time DESC
LIMIT 20;
```

## Practical Decision Guide

**Use `throw` when:**
- Results must be exact and you want failures to be visible.
- You are building data pipelines where silent data loss is unacceptable.
- You prefer to rewrite the query or increase limits rather than accept approximation.

**Use `break` when:**
- The query drives a dashboard where some missing groups are acceptable.
- You want approximate top-N results (the most common groups appear first in the scan).
- Avoiding OOM errors is more important than 100% completeness.

**Use `any` when:**
- You are building approximate analytics where correctness is explicitly traded for memory efficiency.
- You are familiar with the approximation semantics and your consumers understand the data is approximate.

## Conclusion

`group_by_overflow_mode` gives you a safety valve for GROUP BY queries that might exceed memory or row count limits. Configure `max_rows_to_group_by` in user profiles, set `max_bytes_before_external_group_by` to enable disk spill as a first line of defense, and choose `break` or `throw` based on whether your consumers require exact results. Monitor overflow events in `system.query_log` to identify queries that consistently hit limits and need to be rewritten or have their limits raised.
