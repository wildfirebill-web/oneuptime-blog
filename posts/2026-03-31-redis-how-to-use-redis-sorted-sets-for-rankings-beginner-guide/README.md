# How to Use Redis Sorted Sets for Rankings (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sorted Sets, Leaderboards, Rankings, Beginner, Zadd

Description: Learn how to use Redis sorted sets to build leaderboards and ranking systems with fast score updates and rank lookups.

---

## What Is a Redis Sorted Set

A sorted set is a collection of unique members, each associated with a floating-point score. Redis keeps members sorted by score automatically. This makes sorted sets ideal for leaderboards, rankings, priority queues, and time-ordered data.

Key operations are O(log N) - fast even with millions of members.

## Basic Sorted Set Commands

```bash
# ZADD: Add members with scores
ZADD leaderboard 1500 "alice"
ZADD leaderboard 2300 "bob"
ZADD leaderboard 1800 "charlie"
ZADD leaderboard 3100 "diana"

# ZADD multiple at once
ZADD leaderboard 1500 "alice" 2300 "bob" 1800 "charlie" 3100 "diana"

# ZSCORE: Get a player's score
ZSCORE leaderboard "alice"
# "1500"

# ZRANK: Get rank (0-indexed, ascending - lower score = rank 0)
ZRANK leaderboard "alice"
# 0

# ZREVRANK: Get rank (0-indexed, descending - higher score = rank 0)
ZREVRANK leaderboard "diana"
# 0   (diana has the highest score)

# ZINCRBY: Increase score by amount
ZINCRBY leaderboard 500 "alice"
# "2000"  (new score)
```

## Getting Top Players

```bash
# Top 10 players (highest scores first)
ZREVRANGE leaderboard 0 9 WITHSCORES
# 1) "diana"
# 2) "3100"
# 3) "bob"
# 4) "2300"
# ...

# Bottom 10 players
ZRANGE leaderboard 0 9 WITHSCORES

# Players ranked 11 to 20
ZREVRANGE leaderboard 10 19 WITHSCORES
```

## Getting a Player's Rank

```bash
# Get rank starting from 1 instead of 0
ZREVRANK leaderboard "bob"
# 1  (0-indexed, so rank 2 in human terms)
```

In application code, add 1 to convert to 1-based rank:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_player_rank(player):
    rank = r.zrevrank('leaderboard', player)
    if rank is None:
        return None
    return rank + 1  # Convert to 1-based

def get_player_score(player):
    return r.zscore('leaderboard', player)

def add_score(player, points):
    return r.zincrby('leaderboard', points, player)

# Usage
print(get_player_rank('diana'))  # 1
print(get_player_rank('alice'))  # 3
print(get_player_score('bob'))   # 2300.0
```

## Building a Full Leaderboard

```python
def get_leaderboard(top_n=10):
    # Returns list of (player, score) tuples
    results = r.zrevrange('leaderboard', 0, top_n - 1, withscores=True)
    return [
        {'rank': i + 1, 'player': player, 'score': int(score)}
        for i, (player, score) in enumerate(results)
    ]

def get_player_context(player, window=2):
    """Get player's rank and nearby players."""
    rank = r.zrevrank('leaderboard', player)
    if rank is None:
        return None
    start = max(0, rank - window)
    end = rank + window
    nearby = r.zrevrange('leaderboard', start, end, withscores=True)
    return {
        'player': player,
        'rank': rank + 1,
        'nearby': [
            {'rank': start + i + 1, 'player': p, 'score': int(s)}
            for i, (p, s) in enumerate(nearby)
        ]
    }

# Usage
print(get_leaderboard())
# [{'rank': 1, 'player': 'diana', 'score': 3100}, ...]

print(get_player_context('charlie'))
# Shows charlie's rank with 2 players above and below
```

## Time-Based Leaderboards

Use Unix timestamps as scores to create time-ordered sorted sets:

```bash
# Daily leaderboard (reset daily by using a dated key)
ZADD leaderboard:2024-04-01 1500 "alice"

# Weekly leaderboard
ZADD leaderboard:week:14 2300 "bob"
```

For score-based leaderboards that reset daily:

```python
import time

def daily_key():
    return f"leaderboard:daily:{time.strftime('%Y-%m-%d')}"

def add_daily_score(player, points):
    key = daily_key()
    r.zincrby(key, points, player)
    r.expire(key, 86400 * 2)  # Keep for 2 days

def get_daily_top(n=10):
    return r.zrevrange(daily_key(), 0, n - 1, withscores=True)
```

## Players in a Score Range

```bash
# Players with scores between 1000 and 2000
ZRANGEBYSCORE leaderboard 1000 2000 WITHSCORES

# Players with scores above 2000
ZRANGEBYSCORE leaderboard 2000 +inf WITHSCORES

# Count players in range
ZCOUNT leaderboard 1000 2000
```

## Removing Players

```bash
# Remove a single player
ZREM leaderboard "alice"

# Remove players with scores below 1000
ZREMRANGEBYSCORE leaderboard -inf 999

# Remove all but top 100
ZREMRANGEBYRANK leaderboard 0 -101
```

## Summary

Redis sorted sets provide an ideal foundation for leaderboards and rankings, offering O(log N) inserts and rank lookups even with millions of players. The `ZINCRBY` command handles concurrent score updates atomically, while `ZREVRANGE` and `ZREVRANK` make fetching top players and individual ranks trivial. Use dated key names to implement daily, weekly, or monthly leaderboards with automatic expiry.
