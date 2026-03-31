# Redis Data Expiration Strategy Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Expiration, TTL, Best Practice, Cache

Description: Learn how to design effective Redis data expiration strategies - covering TTL selection, expiration jitter, lazy vs active expiration, and monitoring expired key counts.

---

Setting expiration on Redis keys is essential for memory management, but a poorly designed expiration strategy can cause cache stampedes, unexpected data loss, or runaway memory growth. This guide covers practical patterns for expiration in production.

## Always Set TTLs on Cache Keys

Every cache key should have an expiration. Keys without TTLs persist indefinitely and eventually exhaust memory:

```bash
# Set with expiration (seconds)
SET session:abc123 "data" EX 3600

# Set with expiration (milliseconds)
SET rate:user:42 "10" PX 60000

# Set and get expiration atomically
GETEX mykey EX 300
```

Auditing for keys without TTL:

```bash
redis-cli --scan --pattern "*" | while read key; do
    ttl=$(redis-cli TTL "$key")
    if [ "$ttl" -eq -1 ]; then
        echo "No TTL: $key"
    fi
done
```

## Add Jitter to Prevent Cache Stampede

When many keys expire at the same time, concurrent cache misses hammer the database. Add random jitter to TTLs:

```python
import random

BASE_TTL = 3600  # 1 hour

def set_with_jitter(client, key, value, base_ttl=BASE_TTL, jitter_pct=0.1):
    jitter = int(base_ttl * jitter_pct * random.random())
    ttl = base_ttl + jitter  # Spreads expiration over base_ttl to base_ttl + 10%
    client.setex(key, ttl, value)
```

This spreads cache misses across a window rather than creating a thundering herd.

## Use Shorter TTLs for Volatile Data

Match TTL to the rate of change of the underlying data:

```text
User profile (changes rarely):    24 hours
Product inventory (changes often): 5 minutes
Session token:                     30 minutes
Rate limit counter:                1 minute
OTP / verification code:           10 minutes
```

A TTL that is too long serves stale data; one that is too short creates unnecessary database load.

## Understand Active vs. Lazy Expiration

Redis uses two expiration mechanisms:
- **Lazy expiration**: A key is deleted when it is accessed and found to be expired
- **Active expiration**: Redis periodically scans and deletes expired keys (10 times/second by default)

This means expired keys are not always deleted immediately. Do not rely on expiration for real-time cleanup of sensitive data.

```bash
# Check how many keys have been expired
redis-cli INFO stats | grep expired_keys
```

## Use EXPIREAT for Time-Aligned Expiration

When data should expire at a specific wall-clock time (e.g., midnight), use `EXPIREAT`:

```python
import datetime

# Expire at midnight tonight
midnight = datetime.datetime.combine(
    datetime.date.today() + datetime.timedelta(days=1),
    datetime.time.min
)
unix_timestamp = int(midnight.timestamp())

client.expireat("daily-report:2026-03-31", unix_timestamp)
```

## Extend Expiration on Read (Sliding Window)

For session management, extend the TTL every time the session is accessed:

```python
def get_session(session_id: str):
    key = f"session:{session_id}"
    data = client.get(key)
    if data:
        # Extend session by 30 minutes on each access
        client.expire(key, 1800)
        return json.loads(data)
    return None
```

## Monitor Expiration Rate

Sudden spikes in expired keys can indicate a bug or misconfigured TTL:

```bash
redis-cli INFO stats | grep -E "expired_keys|evicted_keys"
```

Set alerts if expired_keys growth rate increases more than 2x baseline.

## Summary

Effective Redis expiration strategy requires setting TTLs on all cache keys, adding jitter to spread expiration across time, and choosing TTLs that match data volatility. Understand that expired keys may persist briefly due to lazy expiration. Use EXPIREAT for wall-clock alignment and sliding expiration for sessions. Monitor expired key counts to catch misconfiguration early.
