# How to Implement a Set with Expiration in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Set, TTL

Description: Implement a Redis set where individual members expire automatically using sorted sets as a TTL-aware set for session tracking and membership lists.

---

Redis sets have no native per-member expiration. The whole key expires, but you cannot set individual member TTLs. If you need members to expire independently - like an active session list or recent-visitor set - use a sorted set with timestamps as scores.

## The Pattern: Sorted Set as Expiring Set

Store the expiration timestamp as the score. Members with a score less than `now` are expired:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def sadd_with_ttl(key: str, member: str, ttl_seconds: int):
    """Add a member that expires after ttl_seconds."""
    expire_at = time.time() + ttl_seconds
    r.zadd(key, {member: expire_at})

def sismember_with_ttl(key: str, member: str) -> bool:
    """Check membership, ignoring expired members."""
    score = r.zscore(key, member)
    if score is None:
        return False
    return score > time.time()

def smembers_active(key: str) -> set:
    """Return only non-expired members."""
    now = time.time()
    # Prune expired members first
    r.zremrangebyscore(key, 0, now)
    return set(r.zrange(key, 0, -1))

def scard_active(key: str) -> int:
    """Count only non-expired members."""
    now = time.time()
    r.zremrangebyscore(key, 0, now)
    return r.zcard(key)
```

## Active Session Tracking

Track which users are currently online with automatic session expiry:

```python
SESSION_TTL = 1800  # 30 minutes

def user_active(user_id: str):
    """Mark a user as active, refreshing their session."""
    sadd_with_ttl("sessions:active", user_id, SESSION_TTL)

def user_logged_out(user_id: str):
    r.zrem("sessions:active", user_id)

def get_online_users() -> set:
    return smembers_active("sessions:active")

def is_user_online(user_id: str) -> bool:
    return sismember_with_ttl("sessions:active", user_id)

def count_online_users() -> int:
    return scard_active("sessions:active")
```

## Recent Visitors Set

Track unique visitors to a page within the last hour:

```python
def record_visitor(page_id: str, visitor_id: str, window_seconds: int = 3600):
    sadd_with_ttl(f"visitors:{page_id}", visitor_id, window_seconds)

def get_recent_visitor_count(page_id: str) -> int:
    return scard_active(f"visitors:{page_id}")
```

## Blocklist with Expiration

Temporarily ban users or IPs:

```python
def block_user(user_id: str, duration_seconds: int, reason: str = ""):
    sadd_with_ttl("users:blocked", user_id, duration_seconds)
    if reason:
        r.setex(f"block:reason:{user_id}", duration_seconds, reason)

def is_blocked(user_id: str) -> bool:
    return sismember_with_ttl("users:blocked", user_id)

def get_block_reason(user_id: str) -> str:
    return r.get(f"block:reason:{user_id}") or ""
```

## Sliding Union of Sets

Combine multiple expiring sets (e.g., merge hourly visitor sets):

```python
def merge_active_sets(keys: list[str], output_key: str, window_seconds: int):
    """Union multiple expiring sets and store result."""
    now = time.time()
    for k in keys:
        r.zremrangebyscore(k, 0, now)
    if keys:
        r.zunionstore(output_key, keys)
        r.expire(output_key, window_seconds)
```

## Periodic Cleanup

Although zadd with scores handles logical expiry, you should prune old members regularly:

```python
def cleanup_expired_members(key: str) -> int:
    """Remove physically expired members. Returns count removed."""
    return r.zremrangebyscore(key, 0, time.time())
```

## Summary

Using a sorted set with expiration timestamps as scores gives you per-member TTL semantics that native Redis sets lack. This pattern is ideal for session tracking, visitor deduplication, and temporary blocklists - call zremrangebyscore before reads to prune expired entries and keep memory usage bounded.
