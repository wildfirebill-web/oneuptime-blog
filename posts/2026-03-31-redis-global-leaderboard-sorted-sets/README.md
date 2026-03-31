# How to Build a Global Leaderboard with Redis Sorted Sets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Leaderboard, Sorted Set, Gaming, Ranking

Description: Build a global leaderboard with Redis Sorted Sets that supports instant rank lookups, score updates, and top-N queries at any scale.

---

Leaderboards are a classic Redis use case. Sorted Sets maintain scores in order automatically, so adding a score update is O(log N) and fetching the top 100 players is O(log N + 100) - fast at any scale.

## Adding and Updating Scores

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

LEADERBOARD_KEY = "leaderboard:global"

def add_score(player_id: str, score: float):
    r.zadd(LEADERBOARD_KEY, {player_id: score})

def increment_score(player_id: str, increment: float) -> float:
    return r.zincrby(LEADERBOARD_KEY, increment, player_id)
```

## Fetching Top Players

```python
def get_top_players(n: int = 10) -> list:
    entries = r.zrevrange(LEADERBOARD_KEY, 0, n - 1, withscores=True)
    return [
        {"rank": i + 1, "player_id": pid, "score": score}
        for i, (pid, score) in enumerate(entries)
    ]
```

## Getting a Player's Rank and Score

```python
def get_player_rank(player_id: str) -> dict:
    rank = r.zrevrank(LEADERBOARD_KEY, player_id)
    score = r.zscore(LEADERBOARD_KEY, player_id)

    if rank is None:
        return {"rank": None, "score": None}

    return {
        "player_id": player_id,
        "rank": rank + 1,  # 0-indexed to 1-indexed
        "score": score,
    }
```

## Players Around a Given Player

Show context - the players just above and below:

```python
def get_surrounding_players(player_id: str, radius: int = 5) -> list:
    rank = r.zrevrank(LEADERBOARD_KEY, player_id)
    if rank is None:
        return []

    start = max(0, rank - radius)
    end = rank + radius

    entries = r.zrevrange(LEADERBOARD_KEY, start, end, withscores=True)
    return [
        {
            "rank": start + i + 1,
            "player_id": pid,
            "score": score,
            "is_self": pid == player_id,
        }
        for i, (pid, score) in enumerate(entries)
    ]
```

## Total Players and Score Distribution

```python
def get_leaderboard_stats() -> dict:
    total = r.zcard(LEADERBOARD_KEY)
    if total == 0:
        return {"total_players": 0}

    # Top score
    top = r.zrevrange(LEADERBOARD_KEY, 0, 0, withscores=True)
    # Median (approximate)
    median_entry = r.zrevrange(LEADERBOARD_KEY, total // 2, total // 2, withscores=True)

    return {
        "total_players": total,
        "top_score": top[0][1] if top else 0,
        "median_score": median_entry[0][1] if median_entry else 0,
    }
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your game backend services and alert when leaderboard API latency spikes.

```bash
redis-cli ZCARD leaderboard:global
redis-cli ZREVRANGE leaderboard:global 0 9 WITHSCORES
```

## Summary

Redis Sorted Sets provide O(log N) score updates and O(log N + M) range queries, making global leaderboards scalable to tens of millions of players on a single instance. ZREVRANK gives you instant rank lookups and ZREVRANGE with offset enables the "players around you" context view essential for engagement.
