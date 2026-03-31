# How to Set preferred_block_size_bytes in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Database, Performance, Configuration, Query, preferred_block_size_bytes

Description: Learn how preferred_block_size_bytes controls the target block size during ClickHouse query reads and how to tune it for optimal I/O and CPU performance.

---

ClickHouse reads data in blocks. The `preferred_block_size_bytes` setting controls the target size (in bytes) for blocks read from MergeTree tables. Tuning this setting affects how much data is read from disk per batch, balancing I/O throughput against CPU pipeline efficiency.

## What preferred_block_size_bytes Controls

When ClickHouse reads from a MergeTree table, it determines how many rows to include in each block by combining two settings:

- `max_block_size` - maximum rows per block
- `preferred_block_size_bytes` - target byte size for a block

The engine picks the smaller of what `max_block_size` allows and what `preferred_block_size_bytes` would produce given the average row size of the columns being read.

Default value: `1000000` (approximately 1 MB)

## Checking the Current Setting

```sql
SELECT value, description
FROM system.settings
WHERE name = 'preferred_block_size_bytes';
```

## Adjusting preferred_block_size_bytes

For narrow tables or queries reading few columns, smaller blocks may improve cache utilization:

```sql
-- Smaller blocks for narrow reads
SET preferred_block_size_bytes = 500000;

SELECT user_id, count() AS events
FROM user_events
WHERE event_date >= today() - 7
GROUP BY user_id;
```

For wide tables or full-table scans, larger blocks reduce overhead per block:

```sql
-- Larger blocks for full scans
SET preferred_block_size_bytes = 4000000;

SELECT
    toStartOfHour(event_time) AS hour,
    count() AS requests,
    avg(response_ms) AS avg_latency
FROM http_logs
WHERE event_date = today()
GROUP BY hour
ORDER BY hour;
```

## Setting It in users.xml

```xml
<profiles>
  <default>
    <preferred_block_size_bytes>1000000</preferred_block_size_bytes>
  </default>
  <analytics>
    <preferred_block_size_bytes>2000000</preferred_block_size_bytes>
  </analytics>
</profiles>
```

## How Block Size Affects Performance

Smaller blocks:
- Reduce memory per pipeline step
- Allow earlier pipelining (results flow faster for LIMIT queries)
- May increase per-block overhead on wide datasets

Larger blocks:
- Improve throughput on full scans
- Reduce function call and pipeline overhead
- Use more memory per stage

## Combining with max_block_size

`preferred_block_size_bytes` works with `max_block_size`:

```sql
-- Set both for fine-grained control
SET max_block_size = 65536;
SET preferred_block_size_bytes = 1000000;

SELECT *
FROM events
WHERE event_date = today()
LIMIT 1000;
```

ClickHouse calculates: `block_rows = min(max_block_size, preferred_block_size_bytes / avg_bytes_per_row)`.

## Observing Block Sizes in Query Logs

After running a query, inspect block statistics:

```sql
SELECT
    query_id,
    read_rows,
    read_bytes,
    formatReadableSize(read_bytes) AS data_read,
    result_rows
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date = today()
ORDER BY event_time DESC
LIMIT 5;
```

## Practical Recommendations

| Scenario | Recommended Value |
|---|---|
| Simple lookups, few columns | 500 KB - 1 MB |
| Wide table full scans | 2 MB - 8 MB |
| LIMIT queries needing fast first result | 256 KB - 512 KB |
| Default general workloads | 1 MB (default) |

## Summary

`preferred_block_size_bytes` is a subtle but impactful tuning knob in ClickHouse. It controls the granularity of reads from MergeTree storage, affecting memory usage, CPU pipeline efficiency, and query latency. For most workloads the default of 1 MB is appropriate, but analytics-heavy or LIMIT-heavy workloads can benefit from tuning this value up or down respectively.
