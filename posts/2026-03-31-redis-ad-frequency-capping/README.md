# How to Implement Ad Frequency Capping with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Advertising, Rate Limit

Description: Implement ad frequency capping with Redis to limit how often a user sees a specific ad within a time window, using sliding window counters and atomic Lua scripts.

---

Ad frequency capping ensures users do not see the same advertisement too many times, which would degrade user experience and waste advertiser budget. Redis is ideal for this because it needs to happen in real time (during bid evaluation, before ad serving) with sub-millisecond latency.

## Types of Frequency Caps

- **Daily cap**: max 3 impressions per day per user per creative
- **Campaign cap**: max 10 impressions total per campaign
- **Hourly cap**: max 1 impression per hour (for high-impact placements)
- **Session cap**: max 2 impressions per browsing session

## Setup

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

FREQ_PREFIX = "freqcap"
```

## Fixed Window Frequency Cap

The simplest approach: use a counter key with a TTL equal to the window duration.

```python
def check_and_increment_fixed(user_id: str, creative_id: str, max_impressions: int,
                               window_seconds: int) -> dict:
    window_start = int(time.time()) - (int(time.time()) % window_seconds)
    key = f"{FREQ_PREFIX}:fixed:{user_id}:{creative_id}:{window_start}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window_seconds + 60)
    count, _ = pipe.execute()

    return {
        "allowed": count <= max_impressions,
        "count": count,
        "max": max_impressions,
        "resets_in": window_seconds - (int(time.time()) % window_seconds)
    }
```

## Sliding Window Frequency Cap

More accurate - uses a sorted set to track exact impression timestamps:

```python
SLIDING_CAP_SCRIPT = """
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window_seconds = tonumber(ARGV[2])
local max_count = tonumber(ARGV[3])
local impression_id = ARGV[4]

local window_start = now - window_seconds

-- Remove impressions outside the window
redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

local count = redis.call('ZCARD', key)

if count >= max_count then
    return cjson.encode({allowed = false, count = count, max = max_count})
end

-- Record this impression
redis.call('ZADD', key, now, impression_id)
redis.call('EXPIRE', key, window_seconds + 60)

return cjson.encode({allowed = true, count = count + 1, max = max_count})
"""

sliding_cap = r.register_script(SLIDING_CAP_SCRIPT)

def check_sliding_cap(user_id: str, creative_id: str, max_impressions: int,
                       window_seconds: int) -> dict:
    key = f"{FREQ_PREFIX}:sliding:{user_id}:{creative_id}"
    impression_id = f"{int(time.time() * 1000)}"

    result = sliding_cap(
        keys=[key],
        args=[time.time(), window_seconds, max_impressions, impression_id]
    )
    return json.loads(result)
```

## Multi-Rule Cap Check

Real campaigns often have multiple rules (daily + campaign total):

```python
def check_all_caps(user_id: str, creative_id: str, campaign_id: str) -> dict:
    RULES = [
        {"name": "daily", "key": f"daily:{user_id}:{creative_id}", "max": 3, "window": 86400},
        {"name": "hourly", "key": f"hourly:{user_id}:{creative_id}", "max": 1, "window": 3600},
        {"name": "campaign", "key": f"campaign:{user_id}:{campaign_id}", "max": 15, "window": 2592000},
    ]

    pipe = r.pipeline()
    for rule in RULES:
        full_key = f"{FREQ_PREFIX}:fixed:{rule['key']}:{int(time.time()) // rule['window']}"
        pipe.get(full_key)
    counts = pipe.execute()

    for rule, count_raw in zip(RULES, counts):
        count = int(count_raw or 0)
        if count >= rule["max"]:
            return {
                "allowed": False,
                "blocked_by": rule["name"],
                "count": count,
                "max": rule["max"]
            }

    # All rules passed - record impression
    pipe2 = r.pipeline()
    for rule in RULES:
        full_key = f"{FREQ_PREFIX}:fixed:{rule['key']}:{int(time.time()) // rule['window']}"
        pipe2.incr(full_key)
        pipe2.expire(full_key, rule["window"] + 60)
    pipe2.execute()

    return {"allowed": True, "rules_checked": len(RULES)}
```

## Bulk Cap Check for RTB

In real-time bidding, you may need to check caps for dozens of ads per request:

```python
def bulk_cap_check(user_id: str, ad_candidates: list[dict]) -> list[dict]:
    now_day = int(time.time()) // 86400
    now_hour = int(time.time()) // 3600

    keys = []
    for ad in ad_candidates:
        keys.append(f"{FREQ_PREFIX}:fixed:daily:{user_id}:{ad['creative_id']}:{now_day}")
        keys.append(f"{FREQ_PREFIX}:fixed:hourly:{user_id}:{ad['creative_id']}:{now_hour}")

    counts = r.mget(keys)
    results = []

    for i, ad in enumerate(ad_candidates):
        daily_count = int(counts[i * 2] or 0)
        hourly_count = int(counts[i * 2 + 1] or 0)

        allowed = daily_count < ad.get("daily_cap", 3) and hourly_count < ad.get("hourly_cap", 1)
        results.append({
            "creative_id": ad["creative_id"],
            "allowed": allowed,
            "daily_count": daily_count,
            "hourly_count": hourly_count
        })

    return results
```

## Summary

Redis ad frequency capping uses fixed-window counters for simple per-day and per-hour limits and sliding-window sorted sets for more accurate sub-hour windows. Multi-rule batch checking with `mget` enables checking all frequency constraints for an entire ad set in a single round trip, making it practical for use inside real-time bidding latency budgets.
