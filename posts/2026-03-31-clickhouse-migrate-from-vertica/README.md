# How to Migrate from Vertica to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Vertica, Migration, Analytics, Data Warehouse, OLAP

Description: A practical guide to migrating analytical workloads from Vertica to ClickHouse, covering schema export, data movement, and query translation.

---

Vertica is a powerful columnar database, but its licensing costs and operational overhead push many teams toward ClickHouse. Both are columnar and use vectorized execution, which makes the migration more straightforward than moving from a row-oriented database.

## Architectural Comparison

| Vertica | ClickHouse |
|---|---|
| Projections (sorted copies) | Projections / ORDER BY |
| Partition pruning | Partition pruning |
| Epoch-based MVCC | Part-based MVCC |
| COPY FROM... | INSERT FROM file/S3 |

## Exporting Schemas from Vertica

Export the DDL using `vsql`:

```bash
vsql -U dbadmin -c "\d my_table" > schema.txt
```

Or use the `EXPORT_OBJECTS` function:

```sql
SELECT EXPORT_OBJECTS('', 'public.events');
```

## Translating a Vertica Schema

Vertica DDL:

```sql
CREATE TABLE events (
    event_id    INT NOT NULL,
    user_id     INT,
    event_type  VARCHAR(64),
    occurred_at TIMESTAMP,
    amount      FLOAT
) ORDER BY occurred_at, user_id
SEGMENTED BY HASH(user_id) ALL NODES;
```

ClickHouse equivalent:

```sql
CREATE TABLE events (
    event_id    UInt64,
    user_id     UInt64,
    event_type  LowCardinality(String),
    occurred_at DateTime,
    amount      Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(occurred_at)
ORDER BY (occurred_at, user_id);
```

## Exporting Data from Vertica

Export to CSV:

```sql
COPY (SELECT * FROM events)
TO '/tmp/events.csv'
DELIMITER ','
NULL AS ''
ENCLOSED BY '"';
```

For large tables, export in partitions:

```sql
COPY (
    SELECT * FROM events
    WHERE occurred_at >= '2024-01-01'
      AND occurred_at  < '2024-02-01'
) TO '/tmp/events_2024_01.csv' DELIMITER ',' ENCLOSED BY '"';
```

## Loading into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT CSVWithNames" \
  < events.csv
```

Or use parallel loading with multiple files:

```bash
ls /tmp/events_*.csv | xargs -P 4 -I{} sh -c \
  "clickhouse-client --query 'INSERT INTO events FORMAT CSVWithNames' < {}"
```

## Query Translation Examples

Vertica percentile:

```sql
SELECT APPROXIMATE_PERCENTILE(duration USING PARAMETERS percentile=0.95) AS p95
FROM requests;
```

ClickHouse equivalent:

```sql
SELECT quantile(0.95)(duration) AS p95
FROM requests;
```

Vertica time series fill:

```sql
SELECT TS_FIRST_VALUE(value, 'CONST') AS filled_value
FROM timeseries_table TIMESERIES ts AS '1 HOUR' OVER (ORDER BY ts);
```

ClickHouse equivalent using `WITH FILL`:

```sql
SELECT
    toStartOfHour(ts) AS hour,
    sum(value) AS total
FROM timeseries_table
GROUP BY hour
ORDER BY hour WITH FILL
    FROM toStartOfHour(now() - INTERVAL 7 DAY)
    TO toStartOfHour(now())
    STEP INTERVAL 1 HOUR;
```

## Vertica Projection vs ClickHouse Projection

Vertica automatically maintains projections. In ClickHouse, define them explicitly:

```sql
ALTER TABLE events
ADD PROJECTION events_by_type (
    SELECT event_type, count(), sum(amount)
    GROUP BY event_type
);

ALTER TABLE events MATERIALIZE PROJECTION events_by_type;
```

## Summary

Migrating from Vertica to ClickHouse is well-suited to CSV or Parquet-based data movement. The core concepts - columnar storage, sort keys, and projections - map directly between the two systems, and most analytical SQL translates with minor function name changes.
