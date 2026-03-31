# How to Monitor Redis Persistence Status and Health

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Persistence, Monitoring, RDB, AOF, DevOps

Description: Learn how to monitor Redis persistence status and health using INFO commands, key metrics, and alerting strategies to prevent data loss.

---

## Understanding Redis Persistence Modes

Redis supports two persistence mechanisms: RDB (snapshotting) and AOF (Append-Only File). Monitoring both is critical to ensure data durability and recovery readiness.

- RDB creates point-in-time snapshots at configured intervals
- AOF logs every write operation for full replay on restart
- Both can be enabled simultaneously for maximum durability

## Checking Persistence Status with INFO

The `INFO persistence` command returns all persistence-related metrics:

```bash
redis-cli INFO persistence
```

Sample output:

```text
# Persistence
loading:0
async_loading:0
current_cow_peak:0
current_cow_size:0
current_cow_size_age:0
current_fork_perc:0.00
current_save_keys_processed:0
current_save_keys_total:0
rdb_changes_since_last_save:3
rdb_bgsave_in_progress:0
rdb_last_save_time:1711900000
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:1
rdb_current_bgsave_time_sec:-1
rdb_saves:10
rdb_last_cow_size:4096
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:2
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_last_cow_size:8192
module_fork_in_progress:0
module_fork_last_cow_size:0
```

## Key Metrics to Monitor

### RDB Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `rdb_changes_since_last_save` | Writes not yet persisted | > 10000 |
| `rdb_last_bgsave_status` | Status of last save | != "ok" |
| `rdb_last_save_time` | Unix timestamp of last save | Age > 3600s |
| `rdb_bgsave_in_progress` | Background save running | Monitor duration |

### AOF Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `aof_last_write_status` | Status of last AOF write | != "ok" |
| `aof_last_bgrewrite_status` | Status of last rewrite | != "ok" |
| `aof_rewrite_in_progress` | Rewrite currently running | Monitor duration |

## Monitoring with a Python Script

```python
import redis
import time
from datetime import datetime

def check_redis_persistence(host='localhost', port=6379):
    r = redis.Redis(host=host, port=port, decode_responses=True)
    info = r.info('persistence')

    print(f"=== Redis Persistence Health Check at {datetime.now()} ===")

    # RDB checks
    rdb_status = info.get('rdb_last_bgsave_status', 'unknown')
    rdb_changes = info.get('rdb_changes_since_last_save', 0)
    last_save = info.get('rdb_last_save_time', 0)
    age_seconds = int(time.time()) - last_save

    print(f"RDB Status: {rdb_status}")
    print(f"Changes since last save: {rdb_changes}")
    print(f"Last save age: {age_seconds}s ago")

    if rdb_status != 'ok':
        print("ALERT: RDB last save failed!")
    if rdb_changes > 10000:
        print(f"WARNING: {rdb_changes} changes not yet persisted")
    if age_seconds > 3600:
        print(f"WARNING: Last save was over an hour ago")

    # AOF checks
    aof_enabled = info.get('aof_enabled', 0)
    if aof_enabled:
        aof_write_status = info.get('aof_last_write_status', 'unknown')
        aof_rewrite_status = info.get('aof_last_bgrewrite_status', 'unknown')
        print(f"AOF Write Status: {aof_write_status}")
        print(f"AOF Rewrite Status: {aof_rewrite_status}")

        if aof_write_status != 'ok':
            print("ALERT: AOF write failed!")
        if aof_rewrite_status != 'ok':
            print("ALERT: AOF rewrite failed!")
    else:
        print("AOF: Disabled")

check_redis_persistence()
```

## Triggering Manual Saves

To force a synchronous save (blocks Redis during save):

```bash
redis-cli SAVE
```

To trigger a background save (non-blocking):

```bash
redis-cli BGSAVE
```

To trigger a background AOF rewrite:

```bash
redis-cli BGREWRITEAOF
```

## Configuring Persistence via redis.conf

```text
# RDB: save every 900s if 1 key changed, every 300s if 10 keys, etc.
save 900 1
save 300 10
save 60 10000

# AOF
appendonly yes
appendfsync everysec

# AOF rewrite trigger
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

## Setting Up Alerting with OneUptime

You can use OneUptime to monitor Redis persistence health continuously:

1. Create a custom monitor that runs the `INFO persistence` check
2. Set alert rules for `rdb_last_bgsave_status != ok` or `aof_last_write_status != ok`
3. Configure notification channels (PagerDuty, Slack, email) for critical alerts
4. Track `rdb_changes_since_last_save` as a metric over time to detect save frequency issues

## Summary

Monitoring Redis persistence requires tracking both RDB and AOF metrics from `INFO persistence`. Key signals are the status fields (`rdb_last_bgsave_status`, `aof_last_write_status`), the age of the last save, and the number of pending changes. Combining automated checks with alerting tools ensures you catch persistence failures before they cause data loss.
