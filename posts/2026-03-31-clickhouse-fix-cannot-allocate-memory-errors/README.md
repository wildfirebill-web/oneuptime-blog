# How to Fix "Cannot allocate memory" Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Memory, Error, Troubleshooting, Performance

Description: Diagnose and resolve "Cannot allocate memory" errors in ClickHouse caused by OS limits, memory overcommit, and large query allocations.

---

The "Cannot allocate memory" error in ClickHouse typically originates from the Linux kernel refusing a memory allocation request. This happens when physical RAM plus swap is exhausted, or when the OS overcommit policy blocks the allocation.

## Identify the Root Cause

Check available memory:

```bash
free -h
cat /proc/meminfo | grep -E "MemAvailable|SwapFree|CommitLimit"
```

Check recent OOM killer activity:

```bash
sudo dmesg | grep -i "oom\|killed\|memory"
journalctl -k | grep -i "out of memory"
```

## Increase Available Memory

Add swap space if RAM is insufficient:

```bash
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Adjust Overcommit Settings

Linux may refuse allocations even when memory appears available:

```bash
# Check current setting (2 = strict, 1 = always allow, 0 = heuristic)
cat /proc/sys/vm/overcommit_memory

# Allow overcommit (useful for ClickHouse)
sudo sysctl -w vm.overcommit_memory=1
echo 'vm.overcommit_memory=1' | sudo tee -a /etc/sysctl.conf
```

## Limit ClickHouse Memory Usage

Configure per-query and server-wide limits in `users.xml`:

```xml
<profiles>
  <default>
    <max_memory_usage>10000000000</max_memory_usage>
    <max_memory_usage_for_all_queries>80000000000</max_memory_usage_for_all_queries>
  </default>
</profiles>
```

In `config.xml`, set total server memory:

```xml
<max_server_memory_usage_to_ram_ratio>0.8</max_server_memory_usage_to_ram_ratio>
```

## Optimize Memory-Heavy Queries

Identify queries consuming the most memory:

```sql
SELECT
    query_id,
    memory_usage,
    query
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 10;
```

Enable memory-spilling for GROUP BY operations:

```sql
SET max_bytes_before_external_group_by = 20000000000;
SET max_bytes_before_external_sort = 20000000000;
```

## Reduce Background Memory Usage

Limit background merge memory:

```xml
<background_pool_size>4</background_pool_size>
<background_merges_mutations_memory_usage_soft_limit>0.5</background_merges_mutations_memory_usage_soft_limit>
```

## Summary

"Cannot allocate memory" errors in ClickHouse stem from OS-level memory exhaustion or overcommit restrictions. Add swap space, adjust the overcommit policy, set per-query memory limits in ClickHouse configuration, and enable external sorting/grouping to spill large operations to disk. Monitor `system.processes` to catch memory-heavy queries before they crash the server.
