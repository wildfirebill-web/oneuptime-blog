# How to Fix 'Cannot allocate memory' Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Memory, OOM, Performance

Description: Diagnose and fix 'Cannot allocate memory' errors in ClickHouse by tuning memory limits, adjusting OS settings, and optimizing query memory usage.

---

"Cannot allocate memory" errors in ClickHouse indicate that either the OS is out of memory, the `vm.overcommit_memory` setting is preventing allocation, or ClickHouse's own memory limits are misconfigured. This guide covers all three cases.

## Identifying the Root Cause

Check the ClickHouse error log first:

```bash
sudo tail -100 /var/log/clickhouse-server/clickhouse-server.err.log | grep -i "allocate\|memory"
```

Also check the OS-level out-of-memory killer:

```bash
sudo dmesg | grep -i "oom\|kill"
journalctl -k | grep -i "oom\|killed"
```

## OS-Level Fix: vm.overcommit_memory

By default, Linux may refuse memory allocation requests even when physical memory is available due to `vm.overcommit_memory=2`. Set it to allow overcommit:

```bash
# Check current value
cat /proc/sys/vm/overcommit_memory

# Set to 1 (always allow overcommit)
sudo sysctl -w vm.overcommit_memory=1

# Make persistent
echo 'vm.overcommit_memory=1' | sudo tee /etc/sysctl.d/99-clickhouse.conf
sudo sysctl --system
```

## OS-Level Fix: Insufficient Swap

If actual physical memory is exhausted, add swap space:

```bash
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Note: Swap helps prevent crashes but ClickHouse performance degrades significantly when using swap.

## ClickHouse Configuration: max_memory_usage

ClickHouse enforces its own memory limits. Check the current limits:

```sql
SELECT name, value
FROM system.settings
WHERE name IN ('max_memory_usage', 'max_memory_usage_for_user', 'max_server_memory_usage');
```

Increase the per-query limit if queries are being killed:

```xml
<clickhouse>
  <profiles>
    <default>
      <max_memory_usage>32212254720</max_memory_usage>
    </default>
  </profiles>
</clickhouse>
```

Set server-wide memory limit to 80% of available RAM:

```xml
<clickhouse>
  <max_server_memory_usage_to_ram_ratio>0.8</max_server_memory_usage_to_ram_ratio>
</clickhouse>
```

## Query-Level Memory Reduction

If a specific query consumes too much memory, optimize it:

```sql
-- Use external aggregation instead of in-memory
SET max_bytes_before_external_group_by = 10000000000;

-- Enable external sort
SET max_bytes_before_external_sort = 10000000000;

-- Check memory used by a query
SELECT
    query_id,
    memory_usage,
    query
FROM system.processes
ORDER BY memory_usage DESC;
```

## Monitoring Memory Pressure

Track memory consumption over time:

```sql
SELECT
    toStartOfMinute(event_time) AS minute,
    max(value) AS max_memory_bytes
FROM system.metric_log
WHERE metric = 'MemoryTracking'
  AND event_time > now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;
```

## Summary

"Cannot allocate memory" errors in ClickHouse are caused by OS overcommit settings (`vm.overcommit_memory`), actual memory exhaustion, or misconfigured ClickHouse memory limits. Fix the OS settings first, then tune `max_server_memory_usage_to_ram_ratio`, and for specific queries enable external group-by and external sort to spill to disk instead of failing with memory errors.
