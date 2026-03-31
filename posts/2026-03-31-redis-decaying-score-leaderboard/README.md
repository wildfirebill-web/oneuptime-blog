# How to Build a Decaying Score Leaderboard with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Leaderboard, Score Decay, Time Decay, Sorted Set

Description: Build a time-decaying leaderboard in Redis where scores fade over time, rewarding recent activity and keeping rankings fresh.

---

In many games and platforms, old achievements should matter less than recent ones. A decaying score leaderboard penalizes inactivity and rewards players who engage consistently. Redis Sorted Sets with recomputed scores implement this elegantly.

## Time-Decay Score Formula

The standard exponential decay formula:

```text
decayed_score = base_score * e^(-lambda * time_since_action_hours)
```

A smaller lambda means slower decay. At lambda=0.1, a score halves roughly every 7 hours.

## Recording Scores with Decay

```python
import redis
import math
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

DECAY_LAMBDA = 0.05  # Slow decay: half-life ~14 hours
LEADERBOARD_KEY = "leaderboard:decaying"

def add_score_with_decay(player_id: str, base_points: float):
    now = time.time()
    # Get existing score and last update time
    existing = r.hget(f"player:meta:{player_id}", "raw_score")
    existing_raw = float(existing or 0)

    # Decay the existing score first, then add new points
    last_update = float(r.hget(f"player:meta:{player_id}", "last_update") or now)
    hours_elapsed = (now - last_update) / 3600
    decayed_existing = existing_raw * math.exp(-DECAY_LAMBDA * hours_elapsed)

    new_raw = decayed_existing + base_points

    pipe = r.pipeline()
    pipe.hset(f"player:meta:{player_id}", mapping={
        "raw_score": new_raw,
        "last_update": now,
    })
    pipe.zadd(LEADERBOARD_KEY, {player_id: new_raw})
    pipe.execute()
```

## Periodic Score Recomputation

Decay all scores on a schedule:

```python
def decay_all_scores():
    now = time.time()
    players = r.zrange(LEADERBOARD_KEY, 0, -1)

    pipe = r.pipeline()
    for player_id in players:
        meta = r.hgetall(f"player:meta:{player_id}")
        if not meta:
            continue
        raw = float(meta.get("raw_score", 0))
        last_update = float(meta.get("last_update", now))
        hours_elapsed = (now - last_update) / 3600
        decayed = raw * math.exp(-DECAY_LAMBDA * hours_elapsed)

        if decayed < 0.01:
            pipe.zrem(LEADERBOARD_KEY, player_id)
        else:
            pipe.zadd(LEADERBOARD_KEY, {player_id: decayed})
            pipe.hset(f"player:meta:{player_id}",
                      mapping={"raw_score": decayed, "last_update": now})

    pipe.execute()
```

## Getting Current Scores with Live Decay

```python
def get_current_score(player_id: str) -> float:
    meta = r.hgetall(f"player:meta:{player_id}")
    if not meta:
        return 0.0
    raw = float(meta.get("raw_score", 0))
    last_update = float(meta.get("last_update", time.time()))
    hours_elapsed = (time.time() - last_update) / 3600
    return raw * math.exp(-DECAY_LAMBDA * hours_elapsed)
```

## Top Players by Decayed Score

```python
def get_top_players(n: int = 10) -> list:
    entries = r.zrevrange(LEADERBOARD_KEY, 0, n - 1, withscores=True)
    return [
        {"rank": i + 1, "player": pid, "score": round(score, 2)}
        for i, (pid, score) in enumerate(entries)
    ]
```

## Monitoring

Schedule decay_all_scores as a background job and monitor its execution with [OneUptime](https://oneuptime.com) to ensure rankings stay fresh.

```bash
redis-cli ZREVRANGE leaderboard:decaying 0 4 WITHSCORES
```

## Summary

Decaying score leaderboards combine per-player metadata Hashes with a Sorted Set holding the current decayed score. Score decay is applied on each new action and periodically via a background job. Lower decay constants (lambda) produce slower score fading, suitable for weekly resets; higher values reward daily engagement.
