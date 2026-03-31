# How to Use Lightweight Deletes in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Delete, Mutation, Performance

Description: Learn how to use ClickHouse Lightweight Deletes for fast row-level deletion without the overhead of traditional heavy mutations.

---

## What Are Lightweight Deletes?

Lightweight Deletes (introduced in ClickHouse 22.8) provide a faster and less resource-intensive way to delete rows compared to traditional `ALTER TABLE ... DELETE` mutations. They work by marking rows as deleted immediately, with the actual data removal happening lazily during background merges.

```sql
-- Traditional heavy mutation (slow, rewrites data parts immediately)
ALTER TABLE events DELETE WHERE user_id = 1001;

-- Lightweight Delete (fast, marks rows as deleted)
DELETE FROM events WHERE user_id = 1001;
```

## Enabling Lightweight Deletes

Lightweight Deletes require the `allow_experimental_lightweight_delete` setting:

```sql
-- Enable for the current session
SET allow_experimental_lightweight_delete = 1;
```

```xml
<!-- /etc/clickhouse-server/users.d/settings.xml -->
<clickhouse>
  <profiles>
    <default>
      <allow_experimental_lightweight_delete>1</allow_experimental_lightweight_delete>
    </default>
  </profiles>
</clickhouse>
```

Note: as of ClickHouse 23.3+, Lightweight Deletes are enabled by default and no longer experimental.

## Basic Syntax

```sql
-- Delete rows matching a condition
DELETE FROM table_name WHERE condition;

-- Examples
DELETE FROM events WHERE user_id = 1001;
DELETE FROM logs WHERE date < today() - 90;
DELETE FROM sessions WHERE expired = 1 AND last_seen < now() - INTERVAL 30 DAY;
```

## How Lightweight Deletes Work Internally

```text
1. ClickHouse adds a hidden _row_exists column to the table
2. DELETE marks matching rows by setting _row_exists = 0
3. Rows are immediately hidden from SELECT queries
4. Background merges physically remove marked rows
5. No immediate data rewrite - very fast execution
```

## Lightweight Delete vs Heavy Mutation Comparison

```sql
-- Heavy mutation approach: rewrites entire data parts immediately
ALTER TABLE events DELETE WHERE date < '2023-01-01';

-- Check mutation progress
SELECT mutation_id, command, is_done, parts_to_do
FROM system.mutations
WHERE database = 'mydb' AND table = 'events'
ORDER BY create_time DESC;

-- Lightweight delete: fast, returns immediately
DELETE FROM events WHERE date < '2023-01-01'
SETTINGS allow_experimental_lightweight_delete = 1;
```

## Practical Examples

Delete GDPR data for a specific user:

```sql
DELETE FROM user_events WHERE user_id = 12345;
DELETE FROM user_sessions WHERE user_id = 12345;
DELETE FROM user_profiles WHERE user_id = 12345;
```

Delete old log data:

```sql
DELETE FROM application_logs
WHERE log_date < today() - 90;
```

Delete based on a subquery (ClickHouse 23.1+):

```sql
DELETE FROM events
WHERE user_id IN (SELECT user_id FROM users WHERE deactivated = 1);
```

## Forcing Physical Deletion

If you need to ensure deleted rows are physically removed for compliance:

```sql
-- Force merge to clean up deleted rows in a specific partition
OPTIMIZE TABLE events PARTITION '202301' FINAL;

-- Optimize the full table (expensive on large tables)
OPTIMIZE TABLE events FINAL;
```

## Performance Characteristics

```sql
-- Check delete performance in query log
SELECT
    query,
    query_duration_ms,
    written_rows,
    written_bytes
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE 'DELETE FROM%'
ORDER BY event_time DESC
LIMIT 10;
```

Lightweight Deletes are typically 10-100x faster than heavy mutations for the initial operation because they do not rewrite data parts immediately.

## Limitations

```text
- Not supported on all table engines (requires MergeTree family)
- Deleted rows still consume storage until a merge occurs
- Very large deletes can still be slow for the background merge
- Not supported on projections in some ClickHouse versions
- Cannot be used in read-only mode
```

## Monitoring Deleted Rows

```sql
-- Check storage consumed by parts with deleted rows
SELECT
    database,
    table,
    sum(rows) AS total_rows,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size
FROM system.parts
WHERE active = 1
  AND database = 'mydb'
GROUP BY database, table;
```

## Summary

ClickHouse Lightweight Deletes use `DELETE FROM table WHERE condition` syntax and are significantly faster than traditional `ALTER TABLE ... DELETE` mutations because they mark rows as deleted immediately without rewriting data parts. Enable them with `allow_experimental_lightweight_delete = 1` on older versions. Physical removal happens during background merges; use `OPTIMIZE TABLE ... FINAL` to force immediate cleanup when required for compliance.
