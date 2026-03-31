# How to Clear Column Data Without Dropping the Column

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, ALTER TABLE, Clear Column

Description: Learn how to reset column data in a ClickHouse partition using ALTER TABLE CLEAR COLUMN IN PARTITION without removing the column from the schema.

---

There are situations where you need to erase the stored values in a column while keeping the column definition intact - for example, clearing personally identifiable information from a specific time range, resetting a computed column before re-materializing it, or purging test data from a partition. ClickHouse provides `ALTER TABLE CLEAR COLUMN IN PARTITION` for exactly this purpose.

## Syntax

```sql
ALTER TABLE table_name
    CLEAR COLUMN column_name IN PARTITION partition_expr;
```

The `partition_expr` must identify a single partition using the same expression used in the table's `PARTITION BY` clause.

## How It Works

`CLEAR COLUMN IN PARTITION` resets every value in the specified column for all rows in the target partition to the column's default value (zero for numeric types, empty string for String, etc.). The column remains in the table schema and continues to exist in all other partitions unchanged.

This operation is a mutation and runs asynchronously, similar to `ALTER TABLE UPDATE` or `ALTER TABLE DELETE`.

## Basic Example

Clear a `user_email` column in the January 2024 partition:

```sql
ALTER TABLE user_events
    CLEAR COLUMN user_email IN PARTITION '2024-01';
```

After the mutation completes, every row in the `2024-01` partition will have `user_email = ''` (the default for String).

## Partitions Identified by Different Expressions

The partition expression depends on the table's `PARTITION BY` clause:

```sql
-- Table partitioned by toYYYYMM(event_time)
ALTER TABLE logs
    CLEAR COLUMN raw_payload IN PARTITION 202403;

-- Table partitioned by toDate(event_time)
ALTER TABLE metrics
    CLEAR COLUMN debug_info IN PARTITION '2024-03-15';

-- Table partitioned by (region, toYYYYMM(event_time))
ALTER TABLE sales
    CLEAR COLUMN notes IN PARTITION ('EU', 202403);
```

## Viewing Partition Names

To find the correct partition expression, query `system.parts`:

```sql
SELECT DISTINCT
    partition,
    partition_id
FROM system.parts
WHERE table = 'user_events' AND database = 'default' AND active = 1
ORDER BY partition;
```

## Monitoring the Mutation

Like all mutations, track progress in `system.mutations`:

```sql
SELECT
    mutation_id,
    command,
    is_done,
    parts_to_do,
    latest_fail_reason
FROM system.mutations
WHERE table = 'user_events'
ORDER BY create_time DESC
LIMIT 5;
```

## Difference from DROP COLUMN

`CLEAR COLUMN` and `DROP COLUMN` serve different purposes:

```sql
-- CLEAR COLUMN: reset values in one partition, keep schema
ALTER TABLE user_events
    CLEAR COLUMN user_email IN PARTITION '2024-01';

-- DROP COLUMN: remove the column from the entire table permanently
ALTER TABLE user_events
    DROP COLUMN user_email;
```

After `CLEAR COLUMN`, the column still appears in `SELECT *`, still receives new values on INSERT, and still exists in all other partitions. After `DROP COLUMN`, the column is gone from the entire table.

## Common Use Cases

### Removing PII from Old Partitions

```sql
-- Clear email addresses from all partitions older than 12 months
ALTER TABLE user_events CLEAR COLUMN user_email IN PARTITION '2023-01';
ALTER TABLE user_events CLEAR COLUMN user_email IN PARTITION '2023-02';
ALTER TABLE user_events CLEAR COLUMN user_email IN PARTITION '2023-03';
```

### Resetting a MATERIALIZED Column Before Re-computation

```sql
-- Clear stale computed values
ALTER TABLE events
    CLEAR COLUMN computed_score IN PARTITION '2024-03';

-- Re-materialize with updated expression
ALTER TABLE events
    MATERIALIZE COLUMN computed_score IN PARTITION '2024-03';
```

### Clearing Debug or Test Data

```sql
ALTER TABLE api_requests
    CLEAR COLUMN debug_headers IN PARTITION '2024-03-31';
```

## Verifying the Result

After the mutation completes, confirm the column was cleared:

```sql
SELECT
    user_email,
    count()
FROM user_events
WHERE toYYYYMM(event_time) = 202401
GROUP BY user_email
LIMIT 5;
-- All rows should show user_email = '' (empty string)
```

## Summary

`ALTER TABLE CLEAR COLUMN IN PARTITION` resets column values to their default within a single partition without altering the column schema or touching other partitions. It is the correct tool when you need to purge data from a column selectively - such as removing PII from old partitions or resetting a computed column before re-materialization - and is far safer than `DROP COLUMN` when the column must be retained.
