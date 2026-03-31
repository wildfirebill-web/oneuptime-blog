# How to Use ALTER TABLE MODIFY ORDER BY in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, ALTER TABLE, ORDER BY, Primary Key

Description: Learn how to change the ORDER BY key of a ClickHouse MergeTree table, understand its constraints, and see how it affects query performance and data.

---

The `ORDER BY` clause of a MergeTree table defines its primary sort key - the physical order in which data is stored in each part. Choosing the right sort key is one of the most impactful decisions when designing a ClickHouse schema. `ALTER TABLE MODIFY ORDER BY` lets you update this key on an existing table, but with important constraints that differ from ordinary column changes.

## Basic Syntax

```sql
ALTER TABLE table_name
    MODIFY ORDER BY (column1, column2, ...);
```

Example - adding a column to an existing sort key:

```sql
-- Original table ordered by (event_date)
ALTER TABLE events
    MODIFY ORDER BY (event_date, user_id);
```

## Constraints on MODIFY ORDER BY

ClickHouse imposes strict rules on what the new ORDER BY key can contain:

1. **New key columns must already exist in the table.** You cannot add a column and modify the ORDER BY in a single statement; add the column first.
2. **The new key can only add columns to the existing key; it cannot remove columns from or reorder the leading key columns** for tables using `PRIMARY KEY` separately. For standard MergeTree where ORDER BY and PRIMARY KEY are identical, you can specify any subset as PRIMARY KEY, but the ORDER BY can be extended freely.
3. **New key columns must not be Nullable.**

Workflow for adding a new column to the sort key:

```sql
-- Step 1: add the new column
ALTER TABLE events
    ADD COLUMN region LowCardinality(String) DEFAULT 'unknown';

-- Step 2: extend the ORDER BY
ALTER TABLE events
    MODIFY ORDER BY (event_date, region, user_id);
```

## How Existing Data Is Affected

`MODIFY ORDER BY` takes effect immediately for all new data parts written after the change. Existing parts retain their old sort order. The new sort order is enforced only after existing parts are merged into new parts as part of the background merge process.

To trigger a merge and apply the new order immediately (useful in development):

```sql
OPTIMIZE TABLE events FINAL;
```

In production, rely on background merges rather than forcing `OPTIMIZE FINAL` on large tables, as it can be resource-intensive.

## Relationship Between ORDER BY and PRIMARY KEY

In MergeTree tables, the `PRIMARY KEY` is a prefix of `ORDER BY`. When you extend ORDER BY, you can keep the same PRIMARY KEY or also update it:

```sql
-- Extend ORDER BY while keeping a smaller PRIMARY KEY for index granularity
ALTER TABLE events
    MODIFY ORDER BY (event_date, region, user_id);

-- The PRIMARY KEY remains (event_date) unless explicitly changed
```

To also update the PRIMARY KEY (must remain a prefix of ORDER BY):

```sql
ALTER TABLE events
    MODIFY PRIMARY KEY (event_date, region);
```

## When to Use MODIFY ORDER BY

- **Improving filter performance** on a column that was not in the original sort key.
- **Optimizing range queries** by adding a time or ID column to reduce the scan range.
- **Fixing a schema design mistake** discovered after initial data load.

Example: a table originally sorted only by date, but most queries also filter by `service_name`:

```sql
ALTER TABLE logs
    ADD COLUMN service_name LowCardinality(String) DEFAULT '';

ALTER TABLE logs
    MODIFY ORDER BY (log_date, service_name, severity);
```

After background merges complete, queries filtering on `service_name` will benefit from the sort order.

## Verifying the New ORDER BY

Confirm the sort key change was applied:

```sql
SELECT
    name,
    engine,
    sorting_key,
    primary_key
FROM system.tables
WHERE name = 'events' AND database = 'default';
```

## Performance Considerations

- Changing ORDER BY does not rewrite data immediately; query performance improves gradually as parts merge.
- On very large tables, triggering `OPTIMIZE TABLE ... FINAL` will read and rewrite all data - plan capacity accordingly.
- A well-chosen ORDER BY reduces the amount of data scanned per query far more effectively than secondary indexes.
- Adding low-cardinality columns near the front of the key provides better skip-index behavior than high-cardinality columns.

## Summary

`ALTER TABLE MODIFY ORDER BY` extends the primary sort key of a MergeTree table without immediately rewriting data. New columns must exist before they can be added to the key, and Nullable columns are not permitted. Existing parts adopt the new order only after background merges rewrite them. Use `system.tables` to verify the updated key, and plan around the gradual rollout of the new sort order on existing data.
