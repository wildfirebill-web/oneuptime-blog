# How to Automate ClickHouse Schema Documentation Generation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Schema, Documentation, Automation, Developer Experience

Description: Automate ClickHouse schema documentation generation by querying system tables and rendering Markdown docs that stay in sync with your database.

---

## The Documentation Drift Problem

Schema documentation written by hand goes stale the moment a column is added or renamed. Automated generation pulls the source of truth directly from ClickHouse system tables, so your docs always match the actual schema.

## What to Document

A useful schema reference includes:

- Table names and engines
- Column names, types, and comments
- Partition and sorting keys
- Row counts and storage sizes

## Querying System Tables for Schema Info

ClickHouse stores schema metadata in `system.tables` and `system.columns`:

```sql
SELECT
    t.database,
    t.name AS table_name,
    t.engine,
    t.partition_key,
    t.sorting_key,
    t.total_rows,
    formatReadableSize(t.total_bytes) AS size
FROM system.tables t
WHERE t.database NOT IN ('system', 'information_schema')
ORDER BY t.database, t.name;
```

Fetch column details:

```sql
SELECT
    database,
    table,
    name AS column_name,
    type,
    comment,
    is_in_partition_key,
    is_in_sorting_key
FROM system.columns
WHERE database NOT IN ('system', 'information_schema')
ORDER BY database, table, position;
```

## Generating Markdown with Python

A short Python script renders the query output as Markdown:

```bash
pip install clickhouse-driver
```

```bash
python3 generate_schema_docs.py --host ch.internal --database analytics --output ./docs/schema
```

The script creates one Markdown file per table. A sample table file looks like:

```text
# events

Engine: ReplacingMergeTree
Partition Key: toYYYYMM(event_time)
Sorting Key: (user_id, event_time)
Rows: 4,200,000,000
Size: 380 GiB

| Column | Type | Comment |
|--------|------|---------|
| event_id | UUID | Unique event identifier |
| user_id | UInt64 | Foreign key to users |
| event_type | LowCardinality(String) | Type of event |
| event_time | DateTime | When the event occurred |
```

## Automating with CI/CD

Add a step to your pipeline that regenerates docs and commits the updated files:

```bash
python3 generate_schema_docs.py --host "${CH_HOST}" --database analytics --output ./docs/schema
git add docs/schema/
git diff --staged --quiet || git commit -m "chore: regenerate schema docs"
git push
```

## Adding Column Comments in ClickHouse

Comments are the primary way to add human-readable descriptions. Set them directly:

```sql
ALTER TABLE events
    COMMENT COLUMN user_id 'Foreign key referencing users.user_id',
    COMMENT COLUMN event_type 'Category of user action, e.g. click, view, purchase';
```

These comments then appear automatically in your generated docs.

## Summary

Automating ClickHouse schema documentation generation by querying `system.tables` and `system.columns` produces accurate, always-current reference docs that eliminate the drift between code and documentation.
