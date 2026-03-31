# How to Use Vertical Format for Human-Readable Output in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Vertical Format, Output Format, Debugging, Human-Readable

Description: Learn how to use the Vertical format in ClickHouse to display query results one column per line, making wide rows and complex data structures easy to read.

---

The Vertical format in ClickHouse displays each row's columns one per line, with the column name shown to the left. This is especially useful when dealing with wide tables, rows with many columns, or values that are difficult to read in a standard tabular format.

## Basic Usage

Append `FORMAT Vertical` to any SELECT query:

```sql
SELECT *
FROM system.tables
WHERE name = 'events'
FORMAT Vertical;
```

Output:
```text
Row 1:
------
database:                    analytics
name:                        events
engine:                      MergeTree
is_temporary:                0
data_paths:                  ['/var/lib/clickhouse/store/abc/def/']
metadata_path:               /var/lib/clickhouse/metadata/analytics/events.sql
uuid:                        550e8400-e29b-41d4-a716-446655440000
comment:
has_own_data:                1
loading_dependencies_tables: []
loading_dependent_tables:    []
```

Each row is presented as a labeled block, making it easy to scan even tables with 20+ columns.

## Using the \G Shortcut in clickhouse-client

In the interactive `clickhouse-client`, you can trigger Vertical format by ending a query with `\G` instead of a semicolon:

```sql
SELECT * FROM system.tables WHERE name = 'events'\G
```

This is equivalent to `FORMAT Vertical` and is familiar to MySQL users.

## Inspecting System Tables

Vertical format is ideal for inspecting ClickHouse system tables that have many columns:

```sql
SELECT *
FROM system.settings
WHERE name = 'max_memory_usage'
FORMAT Vertical;
```

```text
Row 1:
------
name:        max_memory_usage
value:       10000000000
changed:     0
description: Maximum memory usage for processing of single query...
min:
max:
readonly:    0
type:        UInt64
is_obsolete: 0
```

Without Vertical format, this output would be nearly unreadable in a terminal.

## Examining Query Execution Details

When debugging query plans or EXPLAIN output:

```sql
EXPLAIN PLAN
SELECT count()
FROM events
WHERE event_date = today()
FORMAT Vertical;
```

## Comparing with Other Human-Readable Formats

| Format | Best For |
|---|---|
| Vertical | Wide rows, complex inspection, debugging |
| Pretty | Tables that fit in terminal width |
| PrettyCompact | Compact multi-row display |
| PrettyNoEscapes | Piping to non-terminal tools |

## Using Vertical for Debugging

When troubleshooting replication or merge issues, Vertical makes it easy to inspect individual rows from system tables:

```sql
SELECT *
FROM system.replicas
WHERE table = 'events'
FORMAT Vertical;
```

```text
Row 1:
------
database:           analytics
table:              events
engine:             ReplicatedMergeTree
is_leader:          1
can_become_leader:  1
is_readonly:        0
is_session_expired: 0
future_parts:       0
parts_to_check:     0
zookeeper_path:     /clickhouse/tables/events
replica_name:       replica1
...
```

## Summary

The Vertical format transforms ClickHouse's default wide-table output into a readable per-field display. It is invaluable for debugging, inspecting system tables, and reviewing rows with many columns. Use the `\G` shortcut in clickhouse-client for quick access, and keep Vertical format in your toolkit for any time a query returns rows too wide to read horizontally.
