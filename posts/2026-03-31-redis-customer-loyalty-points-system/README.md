# How to Build a Customer Loyalty Points System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, E-Commerce, Sorted Set, Loyalty

Description: Build a customer loyalty points system with Redis - track balances atomically, record transaction history, manage expiring points, and tier customers by accumulated points.

---

Loyalty programs require accurate point balances, full transaction history, and tier-based benefits. Redis handles all of these with atomic counters, Sorted Sets, and Streams.

## Data Model

```text
loyalty:{userId}:balance     -> String: current points balance
loyalty:{userId}:history     -> Stream: point transaction events
loyalty:leaderboard          -> Sorted Set: userId -> total points earned
loyalty:{userId}:expiring    -> Sorted Set: points -> expiry timestamp
```

## Earning Points

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def earn_points(user_id, points, reason, order_id=None, expiry_days=365):
    now = time.time()
    expiry = now + (expiry_days * 86400)

    pipe = r.pipeline()
    pipe.incrby(f"loyalty:{user_id}:balance", points)
    pipe.xadd(f"loyalty:{user_id}:history", {
        "type": "earn",
        "points": str(points),
        "reason": reason,
        "order_id": order_id or "",
        "timestamp": str(now),
    })
    # Track expiring points
    pipe.zadd(f"loyalty:{user_id}:expiring", {str(points): expiry})
    # Update leaderboard
    pipe.zincrby("loyalty:leaderboard", points, user_id)
    pipe.execute()

def get_balance(user_id):
    balance = r.get(f"loyalty:{user_id}:balance")
    return int(balance) if balance else 0
```

## Redeeming Points

```python
def redeem_points(user_id, points, reason, order_id=None):
    key = f"loyalty:{user_id}:balance"
    script = """
    local balance = tonumber(redis.call('GET', KEYS[1]))
    if balance == nil then balance = 0 end
    if balance < tonumber(ARGV[1]) then return -1 end
    return redis.call('DECRBY', KEYS[1], tonumber(ARGV[1]))
    """
    result = r.eval(script, 1, key, points)
    if result == -1:
        return {"success": False, "reason": "insufficient_points"}

    r.xadd(f"loyalty:{user_id}:history", {
        "type": "redeem",
        "points": str(-points),
        "reason": reason,
        "order_id": order_id or "",
        "timestamp": str(time.time()),
    })
    return {"success": True, "new_balance": result}
```

## Transaction History

```python
def get_point_history(user_id, limit=20):
    entries = r.xrevrange(f"loyalty:{user_id}:history", "+", "-", count=limit)
    return [{"id": eid, **fields} for eid, fields in entries]
```

## Customer Tiers

```python
TIERS = [
    ("bronze", 0),
    ("silver", 500),
    ("gold", 2000),
    ("platinum", 10000),
]

def get_customer_tier(user_id):
    total_earned = r.zscore("loyalty:leaderboard", user_id) or 0
    tier = "bronze"
    for tier_name, threshold in TIERS:
        if total_earned >= threshold:
            tier = tier_name
    return tier

def get_leaderboard(limit=10):
    return r.zrevrange("loyalty:leaderboard", 0, limit - 1, withscores=True)
```

## Expiring Old Points

```python
def expire_old_points(user_id):
    now = time.time()
    expired = r.zrangebyscore(f"loyalty:{user_id}:expiring", 0, now, withscores=True)
    if not expired:
        return 0

    total_expired = sum(int(pts) for pts, _ in expired)
    pipe = r.pipeline()
    pipe.decrby(f"loyalty:{user_id}:balance", total_expired)
    pipe.zremrangebyscore(f"loyalty:{user_id}:expiring", 0, now)
    pipe.xadd(f"loyalty:{user_id}:history", {
        "type": "expire",
        "points": str(-total_expired),
        "reason": "points_expired",
        "timestamp": str(now),
    })
    pipe.execute()
    return total_expired
```

## Example Usage

```bash
# Earn 100 points
INCRBY loyalty:user:1:balance 100
XADD loyalty:user:1:history * type earn points 100 reason purchase

# Check balance
GET loyalty:user:1:balance    # 100

# Redeem 50 points
DECRBY loyalty:user:1:balance 50
GET loyalty:user:1:balance    # 50
```

## Summary

Redis atomic INCRBY and DECRBY keep loyalty point balances consistent even under concurrent operations. Streams provide a tamper-resistant transaction log. A Sorted Set leaderboard enables tier computation and top-customer rankings. Point expiry is managed with a TTL-scored Sorted Set that can be processed on a schedule.
