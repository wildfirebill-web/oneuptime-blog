# How to Use Log Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Log Engine, Table Engine, Lightweight, Concurrent, SQL

Description: Learn how to use the Log table engine in ClickHouse for small tables requiring concurrent reads with one file per column and a dedicated mark file.

---

The `Log` engine is a ClickHouse Log-family table engine that stores each column in a separate `.bin` file and uses a shared marks file to track block offsets. Like StripeLog, it supports concurrent reads from multiple threads, but uses separate files per column (like TinyLog) rather than a striped layout. It is suited for small-to-medium tables (up to a few hundred MB) where concurrent read access is needed without the overhead of MergeTree.

## Creating a Log Table

```sql
CREATE TABLE request_log (
    ts         DateTime,
    method     LowCardinality(String),
    path       String,
    status     UInt16,
    duration   UInt32
) ENGINE = Log;
```

## Inserting Data

```sql
INSERT INTO request_log VALUES
    ('2026-03-31 10:00:00', 'GET', '/api/users', 200, 45),
    ('2026-03-31 10:00:01', 'POST', '/api/orders', 201, 123),
    ('2026-03-31 10:00:02', 'GET', '/api/users/42', 404, 12);
```

## Querying Data

```sql
SELECT
    method,
    status,
    count()       AS request_count,
    avg(duration) AS avg_duration_ms
FROM request_log
GROUP BY method, status
ORDER BY request_count DESC;
```

```text
method  status  request_count  avg_duration_ms
GET     200     1              45
POST    201     1              123
GET     404     1              12
```

## Storage Structure

```text
request_log/
  ts.bin          -- DateTime column data
  method.bin      -- LowCardinality(String) data
  path.bin        -- String column data
  status.bin      -- UInt16 data
  duration.bin    -- UInt32 data
  __marks.mrk     -- shared marks for all columns
  sizes.json
```

The marks file stores byte offset pairs for each column at each insert block boundary, enabling concurrent reads that seek to the right offset per thread.

## Concurrent Reads

Multiple SELECT queries can read Log tables simultaneously, unlike TinyLog:

```sql
-- Concurrent sessions, both can run at the same time
SELECT count() FROM request_log WHERE status = 200;
SELECT avg(duration) FROM request_log WHERE method = 'POST';
```

## Characteristics

```text
Feature              Log
Index                None (full scan)
Concurrent reads     Yes
Concurrent writes    No (one writer at a time)
On-disk structure    One .bin per column + shared marks file
ALTER TABLE          Not supported
Deletions            Not supported
Max practical size   ~100 MB - 1 GB
```

## Comparing Log, TinyLog, and StripeLog

```text
Engine     File Structure           Concurrent Reads  Marks File
TinyLog    Column files, no marks   No                None
StripeLog  Single striped file      Yes               Offset file
Log        Column files             Yes               Shared marks file
```

## Use Case: Temporary Analysis Table

```sql
-- Load data into Log table for quick analysis
CREATE TABLE sample_events (
    ts      DateTime,
    type    String,
    user_id UInt64,
    value   Float64
) ENGINE = Log;

INSERT INTO sample_events SELECT ts, type, user_id, value
FROM events SAMPLE 0.01;  -- 1% sample

-- Now analyze concurrently
SELECT type, avg(value) FROM sample_events GROUP BY type;
```

## Limitations

- No partitioning, indexing, or skip indexes.
- Cannot OPTIMIZE or ALTER table structure.
- No distributed/replication support.
- Every query is a full table scan.

## Summary

The Log engine provides per-column files with a shared marks file, enabling concurrent reads that TinyLog cannot offer. Use it for small tables (under 1 GB) that need multi-threaded read access, such as lookup data, test fixtures, or temporary analysis samples. For production analytical workloads, use MergeTree variants.
