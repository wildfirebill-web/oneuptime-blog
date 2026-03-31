# How to Scale MySQL with Vertical Scaling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance, Scaling, Configuration, InnoDB

Description: Learn how to scale MySQL vertically by upgrading hardware and tuning key MySQL parameters to fully utilize additional CPU, memory, and storage resources.

---

## What Is Vertical Scaling?

Vertical scaling (also called "scaling up") means moving MySQL to a larger server with more CPU cores, RAM, and faster storage. Unlike horizontal scaling, vertical scaling does not require application changes or schema redesign. It is often the fastest path to increasing MySQL capacity when you hit performance limits.

However, vertical scaling alone is not enough - you must also reconfigure MySQL to take advantage of the new hardware. A MySQL instance tuned for a 32 GB server will underutilize a 256 GB server if configuration is not updated.

## Memory: InnoDB Buffer Pool

The buffer pool should use 70-80% of available RAM on a dedicated database server. After upgrading to a larger instance:

```ini
[mysqld]
# For a 128 GB server
innodb_buffer_pool_size = 96G
innodb_buffer_pool_instances = 16
```

More buffer pool means more data fits in memory, reducing physical disk reads. Verify the hit rate after the change:

```sql
SELECT
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') AS read_requests,
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') AS disk_reads;
```

A ratio of `disk_reads / read_requests` below 0.01 (1%) indicates a healthy hit rate.

## CPU: Thread and Concurrency Settings

More CPU cores allow higher concurrent query execution. Tune InnoDB thread settings to match:

```ini
[mysqld]
innodb_thread_concurrency = 0
innodb_read_io_threads = 8
innodb_write_io_threads = 8
```

Setting `innodb_thread_concurrency = 0` lets InnoDB manage concurrency automatically, which scales well on servers with 16+ cores.

For the MySQL connection layer, increase `max_connections` to match the higher thread capacity:

```ini
[mysqld]
max_connections = 1000
thread_cache_size = 100
```

## Storage: NVMe and I/O Configuration

Upgrading to NVMe SSDs significantly increases IOPS. Tell MySQL about the improved I/O capacity:

```ini
[mysqld]
innodb_io_capacity = 10000
innodb_io_capacity_max = 20000
innodb_flush_method = O_DIRECT
```

`O_DIRECT` bypasses the OS page cache, which is appropriate when the InnoDB buffer pool already covers your working set. This avoids double-buffering data in both MySQL and the OS.

## Redo Log: Matching Write Throughput

A larger server handles more writes per second. Increase redo log capacity accordingly (MySQL 8.0.30+):

```ini
[mysqld]
innodb_redo_log_capacity = 4G
```

In older versions, use `innodb_log_file_size` and `innodb_log_files_in_group`:

```ini
innodb_log_file_size = 1G
innodb_log_files_in_group = 4
```

## Sort and Join Buffers

With more RAM available, increase per-session sort and join buffers for complex queries:

```ini
[mysqld]
sort_buffer_size = 4M
join_buffer_size = 4M
tmp_table_size = 256M
max_heap_table_size = 256M
```

These are per-connection settings, so set them conservatively relative to `max_connections`.

## Benchmarking After Upgrade

Always benchmark before and after vertical scaling to confirm improvement:

```bash
mysqlslap --user=root --password \
  --concurrency=50 --iterations=3 \
  --auto-generate-sql --auto-generate-sql-load-type=mixed \
  --number-of-queries=10000
```

Compare queries-per-second and average latency between the old and new configuration.

## Summary

Vertical scaling MySQL requires both hardware upgrades and configuration changes to fully exploit the new resources. Increase `innodb_buffer_pool_size` to 70-80% of RAM, raise I/O capacity settings for faster storage, set `innodb_buffer_pool_instances` proportionally, and benchmark with mysqlslap to confirm improvements. Vertical scaling is the simplest path to more MySQL capacity but has physical server limits, beyond which horizontal scaling becomes necessary.
