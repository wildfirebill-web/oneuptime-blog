# How to Track MySQL Buffer Pool Hit Rate

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Monitoring

Description: Calculate and monitor the InnoDB buffer pool hit rate using global status variables and identify when to increase buffer pool size.

---

The InnoDB buffer pool is MySQL's most important memory structure. It caches data and index pages from disk. The buffer pool hit rate measures what percentage of page reads are served from memory rather than disk. A high hit rate means fast queries; a low hit rate means excessive disk I/O.

## Calculating Buffer Pool Hit Rate

```sql
SELECT
  VARIABLE_NAME,
  VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN (
  'Innodb_buffer_pool_reads',
  'Innodb_buffer_pool_read_requests'
);
```

- `Innodb_buffer_pool_read_requests`: total logical read requests
- `Innodb_buffer_pool_reads`: physical reads (disk I/O - cache misses)

**Hit rate formula:**

```sql
SELECT
  ROUND(
    (1 - (
      (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
      (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100, 2
  ) AS buffer_pool_hit_rate_pct;
```

A healthy hit rate is above 99%. Anything below 95% warrants immediate investigation.

## Checking Current Buffer Pool Size

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

```sql
-- Also check actual usage
SELECT
  FORMAT(pool_size * 16384 / 1024 / 1024, 0)  AS pool_size_mb,
  FORMAT(pages_data * 16384 / 1024 / 1024, 0) AS data_cached_mb,
  FORMAT(pages_free * 16384 / 1024 / 1024, 0) AS free_pages_mb
FROM information_schema.INNODB_BUFFER_POOL_STATS;
```

## Monitoring Hit Rate Over Time

Save periodic snapshots in a monitoring table:

```sql
CREATE TABLE IF NOT EXISTS bp_metrics (
  ts              DATETIME DEFAULT CURRENT_TIMESTAMP,
  reads           BIGINT,
  read_requests   BIGINT,
  hit_rate_pct    DECIMAL(5,2)
);

INSERT INTO bp_metrics (reads, read_requests, hit_rate_pct)
SELECT
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'),
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'),
  ROUND((1 - (r.reads / r.requests)) * 100, 2)
FROM (
  SELECT
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') AS reads,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') AS requests
) r;
```

## Prometheus Alert

With `mysqld_exporter`, the hit rate can be computed in PromQL:

```text
1 - (
  rate(mysql_global_status_innodb_buffer_pool_reads[5m]) /
  rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])
)
```

Alert when this drops below 0.95 for 5 minutes.

## Increasing the Buffer Pool

If your hit rate is chronically low and total data size exceeds the buffer pool, increase it:

```text
[mysqld]
innodb_buffer_pool_size = 4G
```

In MySQL 8+, you can change it online without a restart:

```sql
SET GLOBAL innodb_buffer_pool_size = 4294967296;  -- 4 GB
```

## Summary

The InnoDB buffer pool hit rate is computed from `Innodb_buffer_pool_reads` and `Innodb_buffer_pool_read_requests`. Keep it above 99%. If it falls below 95%, check total data size versus buffer pool size, identify hot tables driving cache eviction, and increase the buffer pool if available RAM allows.
