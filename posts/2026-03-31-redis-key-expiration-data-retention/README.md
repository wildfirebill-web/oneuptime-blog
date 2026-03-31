# How to Implement Redis Key Expiration for Data Retention Compliance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Key Expiration, Data Retention, Compliance, GDPR

Description: Learn how to use Redis TTL and key expiration to enforce data retention policies for compliance with GDPR, CCPA, and internal data governance requirements.

---

Data retention compliance requires that personal data is not kept longer than necessary. Redis TTL (time-to-live) is a natural mechanism for enforcing retention limits - keys automatically expire and are deleted without manual intervention.

## Understanding Redis Key Expiration

Redis supports two expiration methods:

```bash
# Set expiry at creation time
SET session:abc123 '{"user_id":1}' EX 3600       # 1 hour
SET session:abc123 '{"user_id":1}' PX 3600000     # 1 hour in milliseconds

# Set expiry on existing key
EXPIRE session:abc123 3600
EXPIREAT session:abc123 1775000000                 # Unix timestamp

# Check remaining TTL
TTL session:abc123      # Returns seconds
PTTL session:abc123     # Returns milliseconds
```

## Mapping Retention Policies to TTLs

Define retention policies as constants in your application:

```python
from enum import IntEnum

class RetentionPolicy(IntEnum):
    SESSION = 3600           # 1 hour
    USER_PREFERENCE = 86400 * 30    # 30 days
    AUDIT_LOG = 86400 * 365  # 1 year
    GDPR_SUBJECT_DATA = 86400 * 365 * 2  # 2 years max
    TEMP_CACHE = 300         # 5 minutes
    ANONYMOUS_ANALYTICS = 86400 * 90  # 90 days

import redis
import json

r = redis.Redis(host='localhost', decode_responses=True)

def store_with_retention(key: str, value: dict, policy: RetentionPolicy):
    """Store a key with a retention policy TTL."""
    r.set(key, json.dumps(value), ex=int(policy))
    r.hset(f"retention:meta:{key}", mapping={
        "policy": policy.name,
        "ttl_seconds": str(int(policy)),
        "created_at": "2026-03-31",
    })
    r.expire(f"retention:meta:{key}", int(policy))
```

## Enforcing GDPR Right to Erasure

When a user requests deletion, expire all their keys immediately:

```python
def gdpr_erase_user(user_id: int, r: redis.Redis):
    """Immediately expire all keys belonging to a user."""
    patterns = [
        f"session:user:{user_id}:*",
        f"pref:user:{user_id}:*",
        f"cache:user:{user_id}:*",
        f"profile:{user_id}",
    ]
    deleted_count = 0
    for pattern in patterns:
        for key in r.scan_iter(pattern):
            r.expire(key, 1)  # Expire in 1 second
            deleted_count += 1

    print(f"Scheduled {deleted_count} keys for immediate expiry for user {user_id}")
    return deleted_count
```

## Scheduled Retention Review

For keys without TTL that should have one, run a periodic audit:

```python
def audit_missing_ttls(r: redis.Redis, prefix: str = "user:"):
    """Find and report keys that are missing TTLs."""
    no_ttl_keys = []
    for key in r.scan_iter(f"{prefix}*"):
        ttl = r.ttl(key)
        if ttl == -1:  # -1 means key exists but has no TTL
            no_ttl_keys.append(key)

    if no_ttl_keys:
        print(f"WARNING: {len(no_ttl_keys)} keys have no TTL set:")
        for key in no_ttl_keys[:10]:  # Print first 10
            print(f"  - {key}")
    else:
        print("All keys have TTLs configured")

    return no_ttl_keys
```

## Setting Default TTLs by Key Pattern

Use a Lua script to apply default TTLs to any key without one:

```lua
-- apply_default_ttl.lua
-- Apply a default TTL to keys matching a pattern that have no expiry
local pattern = ARGV[1]
local default_ttl = tonumber(ARGV[2])
local cursor = "0"
local applied = 0

repeat
    local result = redis.call('SCAN', cursor, 'MATCH', pattern, 'COUNT', 100)
    cursor = result[1]
    local keys = result[2]
    for _, key in ipairs(keys) do
        if redis.call('TTL', key) == -1 then
            redis.call('EXPIRE', key, default_ttl)
            applied = applied + 1
        end
    end
until cursor == "0"

return applied
```

Run it:

```bash
redis-cli --eval apply_default_ttl.lua , "user:pref:*" 2592000
```

## Summary

Redis TTL is your primary tool for data retention compliance. Define retention policies as named constants, always set TTLs at key creation time, and implement a GDPR erasure function that immediately expires user keys on demand. Run periodic audits with `SCAN` to catch keys missing TTLs, and use Lua scripts to apply default TTLs in bulk. This automated approach is more reliable than manual deletion and audit-friendly.
