# How to Use LASTSAVE in Redis to Check Last Save Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Persistence, RDB, Monitoring, Commands

Description: Learn how to use LASTSAVE in Redis to retrieve the Unix timestamp of the last successful RDB snapshot, useful for monitoring data durability and backup freshness.

---

## What Is LASTSAVE

`LASTSAVE` returns the Unix timestamp of the last successful save to disk - either from `BGSAVE`, `SAVE`, or automatic saves triggered by Redis's `save` configuration. It is an O(1) operation and is the primary way to check when your data was last persisted to an RDB snapshot.

```text
LASTSAVE
```

Returns an integer Unix timestamp.

## Basic Usage

```bash
LASTSAVE
# (integer) 1711900800

# Convert to human-readable
date -d @1711900800
# Fri Mar 31 2026 12:00:00 UTC
```

## Checking Save Freshness

```bash
# Check last save time
LASTSAVE
# (integer) 1711897200

# Trigger a new save
BGSAVE

# Wait and check again
LASTSAVE
# (integer) 1711900800  <- updated timestamp
```

## Practical Example in Python

```python
import redis
import datetime
import time

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def check_last_save():
    """Check when data was last saved and how long ago."""
    timestamp = client.lastsave()
    save_time = datetime.datetime.fromtimestamp(timestamp, tz=datetime.timezone.utc)
    age_seconds = time.time() - timestamp
    age_minutes = age_seconds / 60

    print(f"Last save time: {save_time.strftime('%Y-%m-%d %H:%M:%S UTC')}")
    print(f"Age: {age_minutes:.1f} minutes ago ({int(age_seconds)} seconds)")
    return timestamp

check_last_save()
```

## Practical Example in Node.js

```javascript
const { createClient } = require('redis');

const client = createClient();
await client.connect();

async function checkLastSave() {
  const timestamp = await client.lastSave();
  const saveDate = new Date(timestamp * 1000);
  const ageSeconds = (Date.now() / 1000) - timestamp;
  const ageMinutes = ageSeconds / 60;

  console.log(`Last save: ${saveDate.toISOString()}`);
  console.log(`Age: ${ageMinutes.toFixed(1)} minutes`);

  return timestamp;
}

await checkLastSave();
```

## Practical Example in Go

```go
package main

import (
    "context"
    "fmt"
    "time"
    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

    lastSave, err := rdb.LastSave(ctx).Result()
    if err != nil {
        panic(err)
    }

    saveTime := time.Unix(lastSave, 0)
    age := time.Since(saveTime)

    fmt.Printf("Last save: %s\n", saveTime.Format(time.RFC3339))
    fmt.Printf("Age: %.1f minutes\n", age.Minutes())
}
```

## Alerting on Stale Saves

```python
import redis
import time

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def check_save_freshness(max_age_minutes=30):
    """Alert if last save is older than the threshold."""
    timestamp = client.lastsave()
    age_seconds = time.time() - timestamp
    age_minutes = age_seconds / 60

    if age_minutes > max_age_minutes:
        print(f"ALERT: Last save was {age_minutes:.1f} minutes ago!")
        print(f"Threshold: {max_age_minutes} minutes")

        # Trigger a new save
        print("Triggering BGSAVE...")
        client.bgsave()
        return False
    else:
        print(f"Save is fresh: {age_minutes:.1f} minutes old (OK)")
        return True

check_save_freshness(max_age_minutes=30)
```

## Monitoring Save Success

Combine `LASTSAVE` with `INFO persistence` to get the full picture:

```python
import redis
import datetime
import time

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_save_status():
    last_save = client.lastsave()
    info = client.info('persistence')

    return {
        'last_save_timestamp': last_save,
        'last_save_datetime': datetime.datetime.fromtimestamp(
            last_save, tz=datetime.timezone.utc
        ).isoformat(),
        'age_seconds': int(time.time() - last_save),
        'rdb_bgsave_in_progress': bool(info.get('rdb_bgsave_in_progress', 0)),
        'rdb_last_bgsave_status': info.get('rdb_last_bgsave_status', 'unknown'),
        'rdb_last_bgsave_time_sec': info.get('rdb_last_bgsave_time_sec', -1),
        'rdb_changes_since_last_save': info.get('rdb_changes_since_last_save', 0),
    }

status = get_save_status()
for key, value in status.items():
    print(f"{key}: {value}")
```

## Health Check Integration

```python
import redis
import time

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def redis_persistence_health(max_save_age_minutes=60):
    """Health check that includes persistence freshness."""
    try:
        last_save = client.lastsave()
        age_minutes = (time.time() - last_save) / 60

        return {
            'status': 'healthy' if age_minutes <= max_save_age_minutes else 'degraded',
            'last_save_age_minutes': round(age_minutes, 2),
            'threshold_minutes': max_save_age_minutes,
            'persistence_fresh': age_minutes <= max_save_age_minutes,
        }
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}

health = redis_persistence_health()
print(health)
```

## Using LASTSAVE in Backup Verification

```bash
#!/bin/bash
# verify_redis_backup.sh

EXPECTED_MAX_AGE=3600  # 1 hour in seconds

LAST_SAVE=$(redis-cli LASTSAVE)
CURRENT_TIME=$(date +%s)
AGE=$((CURRENT_TIME - LAST_SAVE))

echo "Last save: $(date -d @$LAST_SAVE)"
echo "Age: ${AGE} seconds"

if [ $AGE -gt $EXPECTED_MAX_AGE ]; then
    echo "WARNING: Last save is more than $EXPECTED_MAX_AGE seconds old!"
    echo "Triggering BGSAVE..."
    redis-cli BGSAVE
else
    echo "Last save is fresh (within threshold)"
fi
```

## LASTSAVE and RDB-Only Setups

`LASTSAVE` reflects RDB saves only. If you use AOF for persistence:

```bash
# Check AOF status separately
INFO persistence | grep aof_last
```

For AOF-only setups, `LASTSAVE` still reflects the last time data was written to an RDB file (which may be from server startup if no `BGSAVE` was triggered).

## Summary

`LASTSAVE` returns the Unix timestamp of the last successful Redis RDB snapshot save. Use it to monitor backup freshness, implement persistence health checks, and verify that your data durability SLAs are being met. Combine it with `INFO persistence` for a complete view of both RDB and AOF persistence status.
