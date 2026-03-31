# How to Implement Data Lineage Tracking with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Lineage, Observability, Audit, Metadata

Description: Learn how to implement data lineage tracking in ClickHouse to trace data from source to destination and audit transformations across your pipeline.

---

## What Is Data Lineage

Data lineage tracks the origin, movement, and transformation of data through a pipeline. It answers: where did this row come from, what transformations touched it, and which downstream tables depend on it. Lineage is essential for debugging data quality issues and satisfying audit requirements.

## Lineage Metadata Table

Store lineage records in a dedicated table:

```sql
CREATE TABLE data_lineage
(
    lineage_id      UUID DEFAULT generateUUIDv4(),
    recorded_at     DateTime DEFAULT now(),
    source_table    String,
    target_table    String,
    transformation  LowCardinality(String),
    row_count       UInt64,
    pipeline_run_id String,
    query_hash      String,
    metadata        String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (target_table, recorded_at);
```

## Recording Lineage on Insert

Wrap each pipeline step with lineage recording:

```python
import hashlib
import uuid

def run_transformation(client, source, target, sql, run_id):
    query_hash = hashlib.md5(sql.encode()).hexdigest()
    result = client.execute(sql)
    row_count = client.execute(f"SELECT count() FROM {target}")[0][0]

    client.execute(
        "INSERT INTO data_lineage VALUES",
        [{
            "lineage_id": str(uuid.uuid4()),
            "source_table": source,
            "target_table": target,
            "transformation": "ETL",
            "row_count": row_count,
            "pipeline_run_id": run_id,
            "query_hash": query_hash,
            "metadata": "{}"
        }]
    )
```

## Query System Query Log

ClickHouse's `system.query_log` captures every query with source and target tables:

```sql
SELECT
    query_start_time,
    user,
    query_kind,
    tables,
    read_rows,
    written_rows,
    query
FROM system.query_log
WHERE query_start_time >= now() - INTERVAL 1 HOUR
  AND query_kind = 'Insert'
ORDER BY query_start_time DESC;
```

## Tracing a Row's Origin

Find all pipeline runs that contributed to a target table:

```sql
SELECT
    source_table,
    transformation,
    pipeline_run_id,
    row_count,
    recorded_at
FROM data_lineage
WHERE target_table = 'daily_metrics'
  AND recorded_at >= today() - 7
ORDER BY recorded_at DESC;
```

## Impact Analysis

Find all tables that depend on a source:

```sql
SELECT DISTINCT target_table
FROM data_lineage
WHERE source_table = 'raw_events'
ORDER BY target_table;
```

## Column-Level Lineage

For column-level tracking, store field mappings in metadata:

```sql
INSERT INTO data_lineage
(source_table, target_table, transformation, row_count, pipeline_run_id, query_hash, metadata)
VALUES (
    'raw_events',
    'clean_events',
    'normalize',
    1000000,
    'run-001',
    'abc123',
    '{"columns": {"user_id": "user_id", "ts": "event_time"}}'
);
```

## Summary

Implementing data lineage in ClickHouse combines a custom lineage metadata table, pipeline instrumentation that records source-target-transformation relationships, and system table queries for audit trails. Impact analysis and origin tracing become simple SQL queries, making lineage a practical operational tool rather than just a compliance checkbox.
