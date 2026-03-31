# How to Use MySQL for Time Series Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Time Series, Partition, Index, Performance

Description: Learn how to store, query, and manage time series data in MySQL using partitioning, composite indexes, and range queries to handle high-volume chronological data efficiently.

---

MySQL is not a purpose-built time series database, but it handles moderate time series workloads well when you design the schema correctly. The key techniques are partitioning by time range, building composite indexes with the timestamp as a leading or secondary column, and archiving old data with partition pruning.

## Schema Design for Time Series

Structure time series tables with a mandatory timestamp and an indexed entity identifier:

```sql
CREATE TABLE metrics (
  id          BIGINT UNSIGNED AUTO_INCREMENT,
  entity_id   INT UNSIGNED NOT NULL,
  metric_name VARCHAR(100) NOT NULL,
  value       DOUBLE NOT NULL,
  recorded_at DATETIME(3) NOT NULL,
  PRIMARY KEY (id, recorded_at),
  INDEX idx_entity_time (entity_id, recorded_at)
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(recorded_at)) (
  PARTITION p2026_01 VALUES LESS THAN (TO_DAYS('2026-02-01')),
  PARTITION p2026_02 VALUES LESS THAN (TO_DAYS('2026-03-01')),
  PARTITION p2026_03 VALUES LESS THAN (TO_DAYS('2026-04-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

The primary key must include `recorded_at` when using range partitioning because MySQL requires the partition key to be part of every unique key.

## Inserting Time Series Data

Insert in batches to reduce per-row overhead:

```sql
INSERT INTO metrics (entity_id, metric_name, value, recorded_at) VALUES
  (1, 'cpu_pct',   42.5, '2026-03-31 10:00:00.000'),
  (1, 'cpu_pct',   44.1, '2026-03-31 10:00:01.000'),
  (1, 'mem_mb',  1024.0, '2026-03-31 10:00:01.000'),
  (2, 'cpu_pct',   15.3, '2026-03-31 10:00:01.000');
```

## Querying Recent Data

Range queries on `recorded_at` use the partition pruning to skip irrelevant partitions:

```sql
-- Last 5 minutes of CPU metrics for entity 1
SELECT recorded_at, value
FROM metrics
WHERE entity_id = 1
  AND metric_name = 'cpu_pct'
  AND recorded_at >= NOW() - INTERVAL 5 MINUTE
ORDER BY recorded_at DESC;
```

## Aggregating Time Buckets

Downsampling is common for dashboards. Use `DATE_FORMAT` to group by minute or hour:

```sql
SELECT
  DATE_FORMAT(recorded_at, '%Y-%m-%d %H:%i:00') AS bucket,
  AVG(value)  AS avg_value,
  MAX(value)  AS max_value,
  MIN(value)  AS min_value
FROM metrics
WHERE entity_id = 1
  AND metric_name = 'cpu_pct'
  AND recorded_at >= '2026-03-31 09:00:00'
  AND recorded_at <  '2026-03-31 10:00:00'
GROUP BY bucket
ORDER BY bucket;
```

## Archiving Old Data with Partition Management

The key advantage of partitioning for time series is instant partition drops instead of row-by-row deletes:

```sql
-- Add a new partition for next month
ALTER TABLE metrics REORGANIZE PARTITION p_future INTO (
  PARTITION p2026_04 VALUES LESS THAN (TO_DAYS('2026-05-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Drop an old partition instantly (equivalent to deleting millions of rows)
ALTER TABLE metrics DROP PARTITION p2026_01;
```

Dropping a partition is a metadata operation and completes in milliseconds regardless of how many rows it contains.

## Summary

MySQL handles time series data effectively when you use range partitioning on the timestamp column, add composite indexes with the entity ID and timestamp, and batch inserts. Partition pruning accelerates range queries by skipping irrelevant months, aggregation queries downsample data for dashboards, and old partitions drop instantly without table-wide locking. For very high ingest rates above several hundred thousand rows per second, a purpose-built TSDB is more appropriate.
