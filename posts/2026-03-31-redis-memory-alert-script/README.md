# How to Write a Redis Memory Alert Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Memory, Alert, Monitoring, Script

Description: Write a Redis memory monitoring script that alerts when usage exceeds thresholds, identifies top memory consumers, and sends notifications.

---

Redis runs in memory, and when it fills up, your eviction policy kicks in - or worse, Redis crashes with OOM. A memory alert script monitors usage against configurable thresholds and tells you which key patterns are consuming the most space before you hit the limit.

## Memory Alert Script

```bash
#!/bin/bash
# redis-memory-alert.sh

REDIS_HOST="${REDIS_HOST:-localhost}"
REDIS_PORT="${REDIS_PORT:-6379}"
REDIS_PASSWORD="${REDIS_PASSWORD:-}"
WARN_PCT=75
CRIT_PCT=90
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"

cli_cmd() {
    if [ -n "$REDIS_PASSWORD" ]; then
        redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" -a "$REDIS_PASSWORD" \
            --no-auth-warning "$@"
    else
        redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" "$@"
    fi
}

send_alert() {
    local level="$1"
    local message="$2"
    echo "[$level] $message"
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -s -X POST -H "Content-type: application/json" \
            --data "{\"text\":\"[$level] Redis Memory: $message\"}" \
            "$SLACK_WEBHOOK"
    fi
}

# Get memory info
INFO=$(cli_cmd INFO memory)
USED=$(echo "$INFO" | grep "^used_memory:" | cut -d: -f2 | tr -d '\r ')
MAX=$(echo "$INFO" | grep "^maxmemory:" | cut -d: -f2 | tr -d '\r ')
USED_HUMAN=$(echo "$INFO" | grep "^used_memory_human:" | cut -d: -f2 | tr -d '\r ')
PEAK_HUMAN=$(echo "$INFO" | grep "^used_memory_peak_human:" | cut -d: -f2 | tr -d '\r ')
FRAG_RATIO=$(echo "$INFO" | grep "^mem_fragmentation_ratio:" | cut -d: -f2 | tr -d '\r ')

echo "Used: $USED_HUMAN | Peak: $PEAK_HUMAN | Frag ratio: $FRAG_RATIO"

# Memory percentage check
if [ "$MAX" -gt 0 ]; then
    PCT=$(( USED * 100 / MAX ))
    echo "Memory usage: ${PCT}% (${USED_HUMAN} of $(numfmt --to=iec $MAX))"

    if [ "$PCT" -ge "$CRIT_PCT" ]; then
        send_alert "CRITICAL" "${PCT}% memory used on ${REDIS_HOST}:${REDIS_PORT}"
        exit 2
    elif [ "$PCT" -ge "$WARN_PCT" ]; then
        send_alert "WARNING" "${PCT}% memory used on ${REDIS_HOST}:${REDIS_PORT}"
        exit 1
    fi
else
    echo "No maxmemory configured - running without limit"
fi

# Fragmentation ratio check
FRAG_INT=$(echo "$FRAG_RATIO" | cut -d. -f1)
if [ "$FRAG_INT" -ge 2 ]; then
    send_alert "WARNING" "High fragmentation ratio: $FRAG_RATIO (consider MEMORY PURGE)"
fi

echo "OK: Memory within thresholds"
exit 0
```

## Python Script with Top Key Analysis

Identify which key patterns consume the most memory:

```python
#!/usr/bin/env python3
import redis
import os
from collections import defaultdict

r = redis.Redis(
    host=os.getenv("REDIS_HOST", "localhost"),
    port=int(os.getenv("REDIS_PORT", 6379)),
    password=os.getenv("REDIS_PASSWORD"),
    decode_responses=True,
)

def get_memory_by_prefix(sample_size: int = 1000) -> dict:
    prefix_memory = defaultdict(int)
    prefix_count = defaultdict(int)
    cursor = 0
    sampled = 0

    while sampled < sample_size:
        cursor, keys = r.scan(cursor, count=100)
        for key in keys:
            try:
                mem = r.memory_usage(key) or 0
                prefix = key.split(":")[0]
                prefix_memory[prefix] += mem
                prefix_count[prefix] += 1
                sampled += 1
            except Exception:
                pass
        if cursor == 0:
            break

    return {
        p: {
            "total_bytes": prefix_memory[p],
            "key_count": prefix_count[p],
            "avg_bytes": prefix_memory[p] // max(prefix_count[p], 1),
        }
        for p in sorted(prefix_memory, key=lambda x: -prefix_memory[x])
    }

if __name__ == "__main__":
    info = r.info("memory")
    print(f"Used: {info['used_memory_human']} / Max: {info.get('maxmemory_human', 'unlimited')}")

    print("\nTop memory consumers by key prefix:")
    by_prefix = get_memory_by_prefix(2000)
    for prefix, stats in list(by_prefix.items())[:10]:
        print(f"  {prefix}: {stats['key_count']} keys, "
              f"{stats['total_bytes'] // 1024}KB total, "
              f"{stats['avg_bytes']}B avg")
```

## Cron Schedule

```bash
# Check every 5 minutes
*/5 * * * * /opt/scripts/redis-memory-alert.sh >> /var/log/redis-memory.log 2>&1
```

## Summary

A Redis memory alert script checks used memory against configurable warning and critical thresholds and fires Slack or webhook notifications when limits are approached. The Python companion script identifies which key prefixes consume the most memory using MEMORY USAGE sampling, helping you pinpoint runaway caches or missing TTLs before they cause an outage.
