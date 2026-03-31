# How to Implement Credit Score Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Finance, Credit, Cache, TTL

Description: Cache credit scores in Redis with tiered TTLs and read-through logic to reduce bureau API costs, serve sub-millisecond lookups, and handle provider failures gracefully.

---

Credit score lookups are expensive - bureau APIs charge per call and respond in hundreds of milliseconds. Caching scores in Redis dramatically reduces costs and latency, especially when the same customer applies for multiple products in a short window.

## Caching a Credit Score

Store the score with metadata and a TTL reflecting its validity window:

```python
import redis
import json
import time

r = redis.Redis()

SCORE_TTL = 3600 * 24  # Valid for 24 hours

def cache_credit_score(customer_id, score, bureau, raw_factors):
    key = f"credit_score:{customer_id}"
    payload = {
        "score": score,
        "bureau": bureau,
        "factors": raw_factors,
        "fetched_at": time.time()
    }
    r.set(key, json.dumps(payload), ex=SCORE_TTL)
```

## Read-Through Pattern

Fetch from cache first, fall back to bureau API on miss:

```python
def get_credit_score(customer_id, purpose="loan_application"):
    key = f"credit_score:{customer_id}"
    cached = r.get(key)
    if cached:
        data = json.loads(cached)
        log_cache_hit(customer_id, purpose)
        return data

    # Cache miss - fetch from bureau
    data = fetch_from_bureau(customer_id)
    cache_credit_score(customer_id, data["score"], data["bureau"], data["factors"])
    log_bureau_call(customer_id, purpose)
    return data
```

## Tiered TTLs by Credit Tier

High-risk profiles change more frequently - use shorter TTLs:

```python
def tiered_ttl(score):
    if score >= 750:
        return 3600 * 48   # 2 days for excellent credit
    elif score >= 650:
        return 3600 * 24   # 1 day for good credit
    else:
        return 3600 * 6    # 6 hours for poor credit

def cache_with_tiered_ttl(customer_id, score, bureau, factors):
    key = f"credit_score:{customer_id}"
    payload = json.dumps({"score": score, "bureau": bureau,
                          "factors": factors, "fetched_at": time.time()})
    r.set(key, payload, ex=tiered_ttl(score))
```

## Forced Refresh

Invalidate and re-fetch after a significant credit event (new loan, missed payment):

```python
def invalidate_credit_score(customer_id, reason):
    r.delete(f"credit_score:{customer_id}")
    r.lpush(f"credit_score:{customer_id}:audit",
            json.dumps({"action": "invalidated", "reason": reason, "ts": time.time()}))
    r.ltrim(f"credit_score:{customer_id}:audit", 0, 49)
```

## Negative Cache for Bureau Failures

Cache a "not available" marker when the bureau is down, preventing repeated failing calls:

```python
def cache_unavailable(customer_id, ttl=300):
    r.set(f"credit_score:{customer_id}:unavailable", 1, ex=ttl)

def is_score_unavailable(customer_id):
    return r.exists(f"credit_score:{customer_id}:unavailable") == 1
```

## Compliance Considerations

Track every score access for audit and compliance:

```python
def log_score_access(customer_id, accessor_id, purpose):
    entry = json.dumps({
        "customer_id": customer_id,
        "accessor_id": accessor_id,
        "purpose": purpose,
        "ts": time.time()
    })
    r.rpush("credit_score:access_log", entry)
    r.expire("credit_score:access_log", 86400 * 90)  # 90-day retention
```

## Summary

Redis credit score caching reduces bureau API costs and serves sub-millisecond lookups through read-through logic with tiered TTLs that reflect credit tier stability. Negative caching prevents repeated failed bureau calls during outages, while audit logs maintain the compliance trail required for regulated financial applications.
