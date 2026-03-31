# How to Write a Redis Data Sampling Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Script, Sampling, Analytics, Inspection

Description: Sample a Redis keyspace to analyze key type distribution, TTL coverage, memory usage, and data patterns without scanning every key.

---

A large Redis instance may have tens of millions of keys. Full keyspace scans for analytics are too slow and too disruptive. A sampling script inspects a representative subset to answer questions: what types of keys dominate, what fraction have TTLs, which patterns consume the most memory?

## Bash Sampling Script

```bash
#!/bin/bash
# redis-sample.sh

REDIS_HOST="${REDIS_HOST:-localhost}"
REDIS_PORT="${REDIS_PORT:-6379}"
REDIS_PASSWORD="${REDIS_PASSWORD:-}"
SAMPLE_SIZE="${SAMPLE_SIZE:-1000}"
OUTPUT_FILE="${OUTPUT_FILE:-/tmp/redis-sample.tsv}"

cli_cmd() {
    local args=("-h" "$REDIS_HOST" "-p" "$REDIS_PORT" "--no-auth-warning")
    [ -n "$REDIS_PASSWORD" ] && args+=("-a" "$REDIS_PASSWORD")
    redis-cli "${args[@]}" "$@"
}

log() { echo "[$(date '+%H:%M:%S')] $*"; }

log "Sampling $SAMPLE_SIZE keys from $REDIS_HOST:$REDIS_PORT"
echo -e "key\ttype\tttl\tmemory_bytes" > "$OUTPUT_FILE"

collected=0
cursor=0

while [ "$collected" -lt "$SAMPLE_SIZE" ]; do
    result=$(cli_cmd SCAN "$cursor" COUNT 100)
    cursor=$(echo "$result" | head -1)
    keys=$(echo "$result" | tail -n +2)

    for key in $keys; do
        [ "$collected" -ge "$SAMPLE_SIZE" ] && break
        KEY_TYPE=$(cli_cmd TYPE "$key")
        KEY_TTL=$(cli_cmd TTL "$key")
        KEY_MEM=$(cli_cmd MEMORY USAGE "$key" 2>/dev/null || echo "0")
        echo -e "${key}\t${KEY_TYPE}\t${KEY_TTL}\t${KEY_MEM}" >> "$OUTPUT_FILE"
        collected=$((collected + 1))
    done

    [ "$cursor" = "0" ] && break
done

log "Sampled $collected keys -> $OUTPUT_FILE"
```

## Python Sampling with Analysis

```python
#!/usr/bin/env python3
import redis
import os
from collections import Counter, defaultdict

r = redis.Redis(
    host=os.getenv("REDIS_HOST", "localhost"),
    port=int(os.getenv("REDIS_PORT", 6379)),
    password=os.getenv("REDIS_PASSWORD"),
    decode_responses=True,
)

def sample_keyspace(sample_size: int = 2000) -> dict:
    type_counts = Counter()
    ttl_buckets = Counter()
    prefix_memory = defaultdict(int)
    total_memory = 0
    no_ttl_count = 0
    cursor = 0
    sampled = 0

    while sampled < sample_size:
        cursor, keys = r.scan(cursor, count=200)
        pipe = r.pipeline()
        for key in keys:
            pipe.type(key)
            pipe.ttl(key)
            pipe.memory_usage(key)
        results = pipe.execute()

        for i, key in enumerate(keys):
            key_type = results[i * 3]
            ttl = results[i * 3 + 1]
            mem = results[i * 3 + 2] or 0

            type_counts[key_type] += 1
            total_memory += mem

            prefix = key.split(":")[0] if ":" in key else key[:20]
            prefix_memory[prefix] += mem

            if ttl == -1:
                no_ttl_count += 1
                ttl_buckets["no_expiry"] += 1
            elif ttl < 60:
                ttl_buckets["<1min"] += 1
            elif ttl < 3600:
                ttl_buckets["1min-1h"] += 1
            elif ttl < 86400:
                ttl_buckets["1h-24h"] += 1
            else:
                ttl_buckets[">24h"] += 1

            sampled += 1
            if sampled >= sample_size:
                break

        if cursor == 0:
            break

    return {
        "sampled": sampled,
        "type_distribution": dict(type_counts),
        "ttl_distribution": dict(ttl_buckets),
        "no_ttl_pct": round(no_ttl_count / max(sampled, 1) * 100, 1),
        "avg_memory_bytes": total_memory // max(sampled, 1),
        "top_prefixes_by_memory": sorted(
            prefix_memory.items(), key=lambda x: -x[1]
        )[:10],
    }

if __name__ == "__main__":
    import json
    result = sample_keyspace(int(os.getenv("SAMPLE_SIZE", "2000")))
    print(json.dumps(result, indent=2))
```

## Interpreting Sample Results

```text
Key findings to look for:
- High no_ttl_pct (>50%): many keys lack expiry - possible memory leak
- High string count with large avg_memory: oversized cached values
- Single prefix dominating memory: one feature consuming disproportionate RAM
- Many keys with <1min TTL: high churn - consider write consolidation
```

## Summary

A Redis data sampling script uses SCAN with pipeline batch fetching to collect type, TTL, and memory statistics from a representative key subset without full keyspace scans. The Python version produces structured analytics including TTL distribution, type breakdown, and top memory-consuming prefixes. Regular sampling reveals memory trends and catches configuration issues like missing TTLs before they cause capacity problems.
