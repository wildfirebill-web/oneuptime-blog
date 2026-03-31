# How to Build a Subscriber Profile Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Telecom, Cache

Description: Cache telecom subscriber profiles with Redis to deliver sub-millisecond plan lookups, reduce HSS/HLR database load, and handle millions of active sessions.

---

In telecom networks, every call, data session, and SMS triggers a subscriber profile lookup to validate entitlements, check balances, and enforce policies. These lookups hit central databases (HSS, HLR, BSS) that become bottlenecks at scale. A Redis subscriber profile cache can handle millions of lookups per second with sub-millisecond latency.

## What's in a Subscriber Profile

```python
# Typical subscriber profile fields
SUBSCRIBER_FIELDS = {
    "msisdn": "+15551234567",       # Phone number
    "imsi": "310260000000001",       # SIM identifier
    "plan_id": "unlimited_5g",
    "status": "active",              # active/suspended/terminated
    "data_balance_mb": 50000,
    "voice_balance_minutes": -1,     # -1 = unlimited
    "sms_balance": -1,
    "roaming_enabled": True,
    "5g_enabled": True,
    "last_updated": 1711900000
}
```

## Setup

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

PROFILE_PREFIX = "sub:profile"
SESSION_PREFIX = "sub:session"
CACHE_TTL = 300           # 5 minutes - short enough to pick up plan changes
SESSION_TTL = 3600        # 1 hour for active data sessions
```

## Read-Through Profile Cache

```python
def get_subscriber_profile(msisdn: str) -> dict | None:
    cache_key = f"{PROFILE_PREFIX}:{msisdn}"

    # Try cache first
    cached = r.hgetall(cache_key)
    if cached:
        # Convert string booleans and numbers
        return _deserialize_profile(cached)

    # Cache miss - fetch from BSS/HSS
    profile = fetch_profile_from_hss(msisdn)
    if profile:
        _cache_profile(msisdn, profile)
    return profile

def _cache_profile(msisdn: str, profile: dict):
    cache_key = f"{PROFILE_PREFIX}:{msisdn}"
    serialized = _serialize_profile(profile)
    pipe = r.pipeline()
    pipe.hset(cache_key, mapping=serialized)
    pipe.expire(cache_key, CACHE_TTL)
    pipe.execute()

def _serialize_profile(profile: dict) -> dict:
    return {k: json.dumps(v) if isinstance(v, (dict, list, bool)) else str(v)
            for k, v in profile.items()}

def _deserialize_profile(raw: dict) -> dict:
    result = {}
    for k, v in raw.items():
        try:
            result[k] = json.loads(v)
        except (json.JSONDecodeError, TypeError):
            result[k] = v
    return result

def fetch_profile_from_hss(msisdn: str) -> dict | None:
    # Replace with actual HSS/HLR/BSS lookup
    return {
        "msisdn": msisdn,
        "plan_id": "unlimited_5g",
        "status": "active",
        "data_balance_mb": 50000,
        "voice_balance_minutes": -1,
        "roaming_enabled": True,
        "last_updated": int(time.time())
    }
```

## Real-Time Balance Updates

```python
def deduct_data_usage(msisdn: str, used_mb: float) -> dict:
    cache_key = f"{PROFILE_PREFIX}:{msisdn}"
    profile = get_subscriber_profile(msisdn)

    if not profile:
        raise ValueError(f"Subscriber {msisdn} not found")

    current_balance = float(profile.get("data_balance_mb", 0))
    if current_balance != -1:  # -1 means unlimited
        new_balance = max(0, current_balance - used_mb)
        r.hset(cache_key, "data_balance_mb", str(new_balance))

        if new_balance < 1000:  # Under 1GB warning
            r.publish("sub:alerts", json.dumps({
                "msisdn": msisdn,
                "type": "low_data",
                "balance_mb": new_balance
            }))
        return {"msisdn": msisdn, "balance_mb": new_balance}
    return {"msisdn": msisdn, "balance_mb": -1}
```

## Bulk Profile Lookup

For batch processing (e.g., policy enforcement):

```python
def get_profiles_bulk(msisdns: list[str]) -> dict:
    keys = [f"{PROFILE_PREFIX}:{m}" for m in msisdns]
    pipe = r.pipeline()
    for key in keys:
        pipe.hgetall(key)
    results = pipe.execute()

    profiles = {}
    miss_msisdns = []

    for msisdn, raw in zip(msisdns, results):
        if raw:
            profiles[msisdn] = _deserialize_profile(raw)
        else:
            miss_msisdns.append(msisdn)

    if miss_msisdns:
        pipe2 = r.pipeline()
        for msisdn in miss_msisdns:
            profile = fetch_profile_from_hss(msisdn)
            if profile:
                profiles[msisdn] = profile
                key = f"{PROFILE_PREFIX}:{msisdn}"
                pipe2.hset(key, mapping=_serialize_profile(profile))
                pipe2.expire(key, CACHE_TTL)
        pipe2.execute()

    return profiles
```

## Invalidating on Plan Change

```python
def invalidate_subscriber(msisdn: str, reason: str = "plan_change"):
    deleted = r.delete(f"{PROFILE_PREFIX}:{msisdn}")
    if deleted:
        r.publish("sub:cache_events", json.dumps({
            "event": "invalidated",
            "msisdn": msisdn,
            "reason": reason,
            "ts": int(time.time())
        }))
    return bool(deleted)
```

## Summary

A Redis subscriber profile cache uses hashes for compact storage of multi-field profiles, a read-through pattern to transparently populate from HSS/HLR on miss, and short TTLs (5 minutes) to pick up plan changes promptly. Real-time balance deductions are written directly to the cache hash to keep them accurate between full cache refreshes.
