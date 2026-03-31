# How to Use Redis Sets and Sorted Sets in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, Set, Sorted Set, redis-py

Description: Use Redis sets for unique collections and sorted sets for ranked leaderboards in Python with redis-py, including union, intersection, and range queries.

---

Redis sets store unique, unordered string members. Sorted sets extend this with a floating-point score per member, enabling ranking and range-based queries. Both are essential building blocks for features like tagging, leaderboards, and rate limiting.

## Working with Sets

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Add members
r.sadd("tags:post:42", "python", "redis", "tutorial")

# Check membership
print(r.sismember("tags:post:42", "redis"))   # True
print(r.sismember("tags:post:42", "golang"))  # False

# Count members
print(r.scard("tags:post:42"))  # 3

# Retrieve all members (order not guaranteed)
print(r.smembers("tags:post:42"))  # {'python', 'redis', 'tutorial'}

# Remove a member
r.srem("tags:post:42", "tutorial")

# Pop a random member
print(r.spop("tags:post:42"))
```

## Set Operations

```python
r.sadd("skills:alice", "python", "redis", "kubernetes")
r.sadd("skills:bob", "python", "docker", "kubernetes")

# Intersection - skills both have
print(r.sinter("skills:alice", "skills:bob"))
# {'python', 'kubernetes'}

# Union - all skills combined
print(r.sunion("skills:alice", "skills:bob"))
# {'python', 'redis', 'kubernetes', 'docker'}

# Difference - alice's skills bob doesn't have
print(r.sdiff("skills:alice", "skills:bob"))
# {'redis'}

# Store result of union into a new key
r.sunionstore("skills:team", "skills:alice", "skills:bob")
```

## Working with Sorted Sets

Sorted sets associate a floating-point score with each member:

```python
# Add members with scores (higher score = higher rank)
r.zadd("leaderboard:game1", {
    "alice": 9850,
    "bob": 7200,
    "charlie": 11050,
    "diana": 8900,
})

# Get rank (0-based, lowest score first)
print(r.zrank("leaderboard:game1", "alice"))    # 2 (0=bob, 1=diana, 2=alice)

# Get rank from highest (0 = top player)
print(r.zrevrank("leaderboard:game1", "charlie"))  # 0

# Get top 3 players with scores
top3 = r.zrevrange("leaderboard:game1", 0, 2, withscores=True)
for name, score in top3:
    print(f"{name}: {int(score)}")

# Get score of a member
print(r.zscore("leaderboard:game1", "alice"))  # 9850.0

# Increment score
r.zincrby("leaderboard:game1", 500, "alice")
```

## Range Queries

```python
# Players scoring between 8000 and 10000
players = r.zrangebyscore("leaderboard:game1", 8000, 10000, withscores=True)
for name, score in players:
    print(f"{name}: {int(score)}")

# Count players in score range
print(r.zcount("leaderboard:game1", 8000, 10000))

# Remove members below a score threshold (cleanup old entries)
r.zremrangebyscore("leaderboard:game1", "-inf", 5000)
```

## Sorted Set as a Rate Limiter Window

```python
import time

def is_allowed(user_id: str, limit: int = 10, window: int = 60) -> bool:
    key = f"ratelimit:{user_id}"
    now = time.time()
    pipe = r.pipeline()
    pipe.zremrangebyscore(key, "-inf", now - window)
    pipe.zadd(key, {str(now): now})
    pipe.zcard(key)
    pipe.expire(key, window)
    results = pipe.execute()
    return results[2] <= limit
```

## Summary

Redis sets are ideal for unique collections, tag systems, and set math (union, intersection, difference). Sorted sets add score-based ranking, enabling leaderboards, time-window rate limiters, and priority queues. Both are efficiently supported by redis-py with intuitive method names that mirror Redis commands.
