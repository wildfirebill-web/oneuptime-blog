# How to Monitor Redis Health (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Monitoring, Health Check, INFO, Beginner, Performance, Observability

Description: A beginner-friendly guide to monitoring Redis health using the INFO command, key metrics to watch, and tools for alerting on Redis issues.

---

## Why Monitor Redis?

Redis is fast and simple, but problems can sneak up on you: memory fills up, evictions delete your data, slow commands block other operations, or replication falls behind. Monitoring helps you catch these issues before they impact users.

## The Redis INFO Command

The `INFO` command is your primary monitoring tool. It returns detailed statistics about the Redis server:

```bash
# Get all info
redis-cli INFO

# Get a specific section
redis-cli INFO memory
redis-cli INFO stats
redis-cli INFO replication
redis-cli INFO clients
redis-cli INFO persistence
redis-cli INFO server
```

## Key Metrics to Monitor

### Memory Health

```bash
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"
```

Good values:

```text
used_memory_human: 2.50G      (well below maxmemory)
maxmemory_human: 4.00G        (your configured limit)
mem_fragmentation_ratio: 1.15 (healthy range: 1.0-1.5)
```

Warning signs:

```text
used_memory close to maxmemory -> evictions may start
mem_fragmentation_ratio > 1.5  -> fragmentation issue
mem_fragmentation_ratio < 1.0  -> Redis using swap (bad)
```

### Command Statistics

```bash
redis-cli INFO stats | grep -E "total_commands|rejected_connections|evicted_keys"
```

Key fields:

```text
total_commands_processed  - Total commands since start
rejected_connections      - Connections rejected (max clients hit)
evicted_keys              - Keys evicted due to memory pressure
```

### Key and Connection Counts

```bash
# Total number of keys
redis-cli DBSIZE

# Connected clients
redis-cli CLIENT LIST | wc -l

# Max clients setting
redis-cli CONFIG GET maxclients
```

## Checking for Slow Commands

The slow log records commands that exceed a time threshold:

```bash
# Set threshold: 10ms = 10000 microseconds
redis-cli CONFIG SET slowlog-log-slower-than 10000

# View the 10 most recent slow commands
redis-cli SLOWLOG GET 10

# Count slow commands recorded
redis-cli SLOWLOG LEN

# Reset slow log
redis-cli SLOWLOG RESET
```

## Latency Monitoring

```bash
# Enable latency monitoring
redis-cli CONFIG SET latency-monitor-threshold 50

# Check for latency events
redis-cli LATENCY LATEST

# View history for a specific event type
redis-cli LATENCY HISTORY command
```

## Simple Health Check Script

```bash
#!/bin/bash
HOST="${REDIS_HOST:-localhost}"
PORT="${REDIS_PORT:-6379}"

PING=$(redis-cli -h "$HOST" -p "$PORT" PING 2>&1)
if [ "$PING" != "PONG" ]; then
  echo "CRITICAL: Redis not responding"
  exit 2
fi

EVICTED=$(redis-cli -h "$HOST" -p "$PORT" INFO stats | grep "evicted_keys:" | cut -d: -f2 | tr -d '\r')
if [ "${EVICTED:-0}" -gt "0" ]; then
  echo "WARNING: $EVICTED keys evicted"
else
  echo "OK: No evictions"
fi

echo "Redis health check complete"
```

## Python Health Check

```python
import redis
import sys

def check_redis_health(host='localhost', port=6379):
    try:
        r = redis.Redis(host=host, port=port, decode_responses=True)
        r.ping()
        print("OK: Redis is responding")

        info = r.info('memory')
        used = info['used_memory']
        maxmem = info['maxmemory']

        if maxmem > 0:
            pct = (used / maxmem) * 100
            label = "WARNING" if pct > 85 else "OK"
            print(f"{label}: Memory {pct:.1f}% ({info['used_memory_human']} / {info['maxmemory_human']})")

        frag = info['mem_fragmentation_ratio']
        label = "WARNING" if frag > 1.5 or frag < 1.0 else "OK"
        print(f"{label}: Fragmentation ratio {frag:.2f}")

        stats = r.info('stats')
        evicted = stats['evicted_keys']
        if evicted > 0:
            print(f"WARNING: {evicted} keys evicted")

        repl = r.info('replication')
        print(f"INFO: Role={repl['role']}, Replicas={repl.get('connected_slaves', 0)}")

        return True

    except redis.exceptions.ConnectionError as e:
        print(f"CRITICAL: Cannot connect: {e}")
        return False

if __name__ == '__main__':
    sys.exit(0 if check_redis_health() else 1)
```

## Real-Time Stats with redis-cli

```bash
# Live stats dashboard (updates every second)
redis-cli --stat

# Sample output:
# keys       mem      clients blocked requests  connections
# 1234       2.50M    10      0       1234      42
```

## Key Metrics and Alert Thresholds

```text
Metric                          Warning     Critical
------                          -------     --------
Memory usage %                  85%         95%
mem_fragmentation_ratio         > 1.5       > 2.0
evicted_keys rate               > 0         > 100/min
rejected_connections            > 0         > 10
replication lag (seconds)       > 5         > 30
slowlog entries per minute      > 5         > 20
```

## Checking Replication Health

```bash
redis-cli INFO replication

# Key fields:
# role: master or replica
# connected_slaves: number of connected replicas
# master_repl_offset: primary's replication offset
# slave_repl_offset: replica's offset (compare to detect lag)
```

## Summary

Redis health monitoring revolves around the INFO command, providing memory usage, eviction counts, command statistics, and replication status. Monitor memory against maxmemory to prevent eviction surprises, check mem_fragmentation_ratio for memory health, and review the slow log to catch blocking commands. A simple health check script running every minute catches most Redis issues before users notice.
