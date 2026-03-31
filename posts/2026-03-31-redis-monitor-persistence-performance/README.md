# How to Monitor Redis Persistence Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Persistence, Monitoring, Observability

Description: Monitor Redis RDB and AOF persistence health using INFO persistence metrics, latency tracking, and alerting to catch problems before they cause data loss.

---

Redis persistence - both RDB snapshots and AOF - can fail silently or degrade performance without obvious errors. Regular monitoring of persistence metrics helps you catch issues before they cause data loss or availability problems.

## Key Metrics from INFO persistence

```bash
redis-cli INFO persistence
```

The most important fields:

```text
rdb_changes_since_last_save:4827
rdb_last_save_time:1711900800
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:3
rdb_current_bgsave_time_sec:-1
rdb_last_cow_size:8388608

aof_enabled:1
aof_current_size:134217728
aof_base_size:67108864
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0
```

## Critical Alerts to Set Up

### Alert: RDB save failing

```bash
STATUS=$(redis-cli INFO persistence | grep rdb_last_bgsave_status | cut -d: -f2 | tr -d '\r ')
if [ "$STATUS" != "ok" ]; then
    echo "CRITICAL: Redis RDB save failed - status: $STATUS"
fi
```

### Alert: No save in a long time

```bash
LAST_SAVE=$(redis-cli LASTSAVE)
NOW=$(date +%s)
AGE=$((NOW - LAST_SAVE))
if [ "$AGE" -gt 7200 ]; then
    echo "WARNING: Redis has not saved in $((AGE/3600)) hours"
fi
```

### Alert: High AOF delayed fsync

```bash
DELAYED=$(redis-cli INFO persistence | grep aof_delayed_fsync | cut -d: -f2 | tr -d '\r ')
if [ "$DELAYED" -gt 100 ]; then
    echo "WARNING: AOF delayed fsync count: $DELAYED"
fi
```

### Alert: Large changes since last save

```bash
CHANGES=$(redis-cli INFO persistence | grep rdb_changes_since_last_save | cut -d: -f2 | tr -d '\r ')
if [ "$CHANGES" -gt 1000000 ]; then
    echo "WARNING: $CHANGES unsaved changes since last RDB snapshot"
fi
```

## Monitoring Save Duration

Long BGSAVE durations indicate disk contention or large datasets:

```bash
redis-cli INFO persistence | grep rdb_last_bgsave_time_sec
```

Track this over time. If BGSAVE duration starts increasing without a corresponding dataset size increase, investigate disk I/O:

```bash
iostat -x 1 5
```

## Monitoring Copy-on-Write Overhead

```bash
redis-cli INFO persistence | grep rdb_last_cow_size
```

High COW overhead means many writes are occurring during snapshots. In bytes:

- < 100 MB: normal
- 100 MB - 500 MB: elevated, consider scheduling saves during off-peak hours
- > 500 MB: problematic, investigate write rate or dataset size

## AOF Buffer Monitoring

```bash
watch -n 2 "redis-cli INFO persistence | grep -E 'aof_buffer|aof_rewrite_buffer|aof_pending'"
```

`aof_rewrite_buffer_length` growing continuously means the rewrite child cannot keep up with incoming writes. Check:

```bash
redis-cli INFO stats | grep instantaneous_ops_per_sec
```

## Using Redis Latency Monitoring

Enable latency tracking for persistence events:

```bash
redis-cli CONFIG SET latency-monitor-threshold 100
redis-cli LATENCY LATEST
```

```text
1) 1) "aof-stat"
   2) (integer) 1711900800
   3) (integer) 120
   4) (integer) 250
2) 1) "bgsave"
   2) (integer) 1711900600
   3) (integer) 3000
   4) (integer) 3000
```

The third field is the latest latency (ms) and the fourth is the maximum observed.

## Prometheus Metrics Export

If you use the redis_exporter, these metrics are available:

```text
redis_rdb_last_bgsave_status{...} 1
redis_rdb_last_save_timestamp_seconds{...} 1.7119e+09
redis_rdb_last_bgsave_duration_sec{...} 3
redis_aof_enabled{...} 1
redis_aof_current_size_bytes{...} 1.34e+08
redis_aof_delayed_fsync_total{...} 0
```

## Summary

Monitor Redis persistence health by tracking `rdb_last_bgsave_status`, time since `LASTSAVE`, `rdb_last_cow_size`, and `aof_delayed_fsync` from `INFO persistence`. Set up automated alerts for failed saves, stale save timestamps, and high delayed fsync counts. Use `LATENCY LATEST` to track persistence-related latency spikes and correlate them with application response time degradation.
