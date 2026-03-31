# Redis Monitoring and Alerting Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Monitoring, Alerting, Observability, Production

Description: Learn what Redis metrics to monitor, how to set up meaningful alerts, and how to avoid alert fatigue while catching real production issues before they escalate.

---

Monitoring Redis effectively means knowing which metrics matter, what thresholds indicate problems, and how to alert on them without drowning in noise. This guide covers the key metrics, recommended thresholds, and alerting strategies.

## Core Metrics to Monitor

Start with these metrics from `redis-cli INFO all`:

```bash
redis-cli INFO all | grep -E "used_memory_human|connected_clients|keyspace_hits|keyspace_misses|instantaneous_ops_per_sec|rdb_last_bgsave_status|aof_last_rewrite_status|blocked_clients|evicted_keys|rejected_connections"
```

Key metrics and what they indicate:

```text
Metric                       | Alert Threshold
-----------------------------|----------------
used_memory / maxmemory      | > 85%
connected_clients            | > 80% of maxclients
keyspace_miss_rate           | > 50% (cache hit rate below 50%)
evicted_keys                 | > 0 (unexpected evictions)
rejected_connections         | > 0 (connection limit hit)
blocked_clients              | > 10 for extended periods
instantaneous_ops_per_sec    | Drop > 30% from baseline
```

## Calculate Cache Hit Rate

The hit rate is one of the most important Redis health indicators:

```bash
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
```

```python
hits = int(info['keyspace_hits'])
misses = int(info['keyspace_misses'])
hit_rate = hits / (hits + misses) * 100 if (hits + misses) > 0 else 0
```

A healthy cache typically has a hit rate above 90%. Falling below 70% warrants investigation.

## Monitor Replication Lag

For replicated setups, track replication lag from the primary's INFO output:

```bash
redis-cli INFO replication | grep -E "master_repl_offset|slave_repl_offset|lag"
```

Alert if `master_replid` changes unexpectedly (failover occurred) or if `repl_backlog_active` is 0 when replicas are connected.

## Use Latency Monitoring

Enable latency monitoring to detect slow operations:

```text
# redis.conf
latency-monitor-threshold 100
```

```bash
redis-cli LATENCY LATEST
redis-cli LATENCY HISTORY event
```

Alert if any latency event exceeds 100ms.

## Set Up Slow Log Alerts

The slow log captures commands exceeding a threshold:

```text
# redis.conf
slowlog-log-slower-than 10000  # microseconds (10ms)
slowlog-max-len 128
```

```bash
redis-cli SLOWLOG GET 10
```

Alert if the slow log grows rapidly - it means commands are degrading.

## Memory Alert Thresholds

```text
used_memory > 70% of maxmemory: Warning
used_memory > 85% of maxmemory: Critical
mem_fragmentation_ratio > 1.5: Warning
mem_fragmentation_ratio > 2.0: Critical
```

## Persistence Health Alerts

```bash
redis-cli INFO persistence | grep -E "rdb_last_bgsave_status|aof_last_rewrite_status|rdb_last_save_time"
```

Alert conditions:
- `rdb_last_bgsave_status: err` - last RDB save failed
- `aof_last_rewrite_status: err` - last AOF rewrite failed
- Last successful save more than 24 hours ago

## Avoid Alert Fatigue

Design alerts at two levels:

```text
WARNING: Requires attention within business hours
CRITICAL: Requires immediate action (page on-call)
```

Use rate-of-change alerts rather than static thresholds for volatile metrics like ops/sec. Alert on trends, not momentary spikes.

## Summary

Effective Redis monitoring focuses on memory utilization, cache hit rate, connection counts, replication health, and persistence status. Set two-tier alert thresholds to distinguish urgent from non-urgent issues, and use the slow log and latency monitoring to catch performance degradation early. Consistent monitoring prevents surprises and gives your team time to respond before issues become outages.
