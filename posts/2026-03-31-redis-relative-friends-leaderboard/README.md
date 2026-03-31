# How to Build a Relative Leaderboard (Friends Only) with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Leaderboard, Social, Friends, Sorted Set

Description: Build a friends-only relative leaderboard in Redis that shows rankings within a user's social circle using Sorted Set intersections.

---

A global leaderboard can be discouraging when you are ranked 5,847th. A friends leaderboard shows your rank among people you actually know, dramatically improving engagement. Redis Sorted Set operations make friend-scoped rankings fast.

## Data Model

Store each user's score in the global leaderboard and their friend graph as a Set:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def add_friend(user_id: str, friend_id: str):
    r.sadd(f"friends:{user_id}", friend_id)
    r.sadd(f"friends:{friend_id}", user_id)

def remove_friend(user_id: str, friend_id: str):
    r.srem(f"friends:{user_id}", friend_id)
    r.srem(f"friends:{friend_id}", user_id)

def update_score(user_id: str, score: float):
    r.zadd("leaderboard:global", {user_id: score})
```

## Building the Friend Leaderboard

```python
import time

def get_friends_leaderboard(user_id: str, n: int = 20) -> list:
    friends = r.smembers(f"friends:{user_id}")
    # Include the user themselves
    members = list(friends) + [user_id]

    # Create a temporary sorted set containing only friends
    temp_key = f"leaderboard:friends:{user_id}:temp"
    pipe = r.pipeline()

    # Copy scores for each friend from global leaderboard
    for member in members:
        score = r.zscore("leaderboard:global", member)
        if score is not None:
            pipe.zadd(temp_key, {member: score})

    pipe.expire(temp_key, 60)  # Cache for 1 minute
    pipe.execute()

    entries = r.zrevrange(temp_key, 0, n - 1, withscores=True)
    return [
        {
            "rank": i + 1,
            "user_id": uid,
            "score": score,
            "is_self": uid == user_id,
        }
        for i, (uid, score) in enumerate(entries)
    ]
```

## Efficient ZINTERSTORE Approach

For large friend lists, use ZINTERSTORE with a friends bitmap:

```python
def get_friends_leaderboard_v2(user_id: str, n: int = 20) -> list:
    friends_key = f"friends:{user_id}"
    temp_scores_key = f"lb:friends_scores:{user_id}"

    # Build a temporary sorted set from the friend set
    friends = r.smembers(friends_key)
    friends.add(user_id)

    pipe = r.pipeline()
    # Bulk-fetch scores
    for f in friends:
        score = r.zscore("leaderboard:global", f)
        if score is not None:
            pipe.zadd(temp_scores_key, {f: score})
    pipe.expire(temp_scores_key, 120)
    pipe.execute()

    entries = r.zrevrange(temp_scores_key, 0, n - 1, withscores=True)
    return [
        {"rank": i + 1, "user_id": uid, "score": s}
        for i, (uid, s) in enumerate(entries)
    ]
```

## Getting Your Rank Among Friends

```python
def get_friend_rank(user_id: str) -> int:
    board = get_friends_leaderboard(user_id)
    for entry in board:
        if entry["is_self"]:
            return entry["rank"]
    return -1
```

## Monitoring

Monitor the friends-leaderboard API endpoint with [OneUptime](https://oneuptime.com) to catch latency increases as friend lists grow.

```bash
redis-cli SMEMBERS friends:user_123 | wc -l
```

## Summary

Friends-only leaderboards require composing a view from the global Sorted Set filtered by the user's friend Set. Caching the composite temporary key for 60 seconds balances freshness with performance. For users with very large friend lists (over 1,000), consider pre-computing friend leaderboards asynchronously on score updates.
