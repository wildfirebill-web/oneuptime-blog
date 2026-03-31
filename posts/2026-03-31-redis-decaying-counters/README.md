# How to Implement Decaying Counters with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, Decay, Trending, Analytics

Description: Build decaying counters in Redis that weight recent events more heavily than older ones for trending algorithms and relevance scoring.

---

A raw event count treats a post from three years ago the same as one from ten minutes ago. Decaying counters apply a time-based weight, making recent activity more influential. This is the core of trending algorithms, relevance scores, and recency-weighted leaderboards.

## Exponential Decay Formula

The most common decay function reduces a score by a factor over time:

```text
score(t) = initial_score * e^(-lambda * elapsed_time)
```

Higher lambda means faster decay. At lambda = 0.1 and 24 hours elapsed:
```text
score = 100 * e^(-0.1 * 24) = 100 * 0.091 = 9.1
```

## Score on Write Approach

The simplest implementation recalculates the decayed score whenever a new event arrives:

```python
import redis
import time
import math

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

DECAY_LAMBDA = 0.1  # per hour

def add_score(item_id: str, points: float = 1.0):
    key = f"decay:{item_id}"
    now = time.time()

    # Get current score and last update time
    pipe = r.pipeline()
    pipe.hget(key, "score")
    pipe.hget(key, "updated_at")
    current_score, last_updated = pipe.execute()

    current_score = float(current_score or 0)
    last_updated = float(last_updated or now)

    # Apply decay to existing score
    elapsed_hours = (now - last_updated) / 3600
    decayed = current_score * math.exp(-DECAY_LAMBDA * elapsed_hours)

    # Add new points
    new_score = decayed + points

    r.hset(key, mapping={
        "score": new_score,
        "updated_at": now,
        "item_id": item_id,
    })
    return new_score
```

## Maintaining a Decayed Leaderboard

To rank items by decayed score, update a sorted set alongside the hash:

```python
def add_score_with_ranking(item_id: str, points: float = 1.0):
    new_score = add_score(item_id, points)
    r.zadd("trending", {item_id: new_score})
    return new_score

def get_trending(n: int = 10) -> list:
    results = r.zrevrangebyscore("trending", "+inf", "-inf",
                                  start=0, num=n, withscores=True)
    return [{"id": item_id, "score": round(score, 2)} for item_id, score in results]
```

## Periodic Decay Job

Scores only decay accurately when they are updated. Run a background job to apply decay to stale items:

```python
def apply_decay_to_all():
    now = time.time()
    # Get all items in the leaderboard
    items = r.zrangebyscore("trending", "-inf", "+inf", withscores=True)

    for item_id, _ in items:
        key = f"decay:{item_id}"
        data = r.hgetall(key)
        if not data:
            continue

        last_updated = float(data.get("updated_at", now))
        current_score = float(data.get("score", 0))
        elapsed_hours = (now - last_updated) / 3600
        new_score = current_score * math.exp(-DECAY_LAMBDA * elapsed_hours)

        if new_score < 0.01:
            r.zrem("trending", item_id)
            r.delete(key)
        else:
            r.hset(key, mapping={"score": new_score, "updated_at": now})
            r.zadd("trending", {item_id: new_score})
```

## Half-Life Decay

Express decay as a half-life (time to halve the score) rather than lambda:

```python
def halflife_decay(score: float, elapsed_seconds: float, half_life_seconds: float) -> float:
    return score * math.pow(0.5, elapsed_seconds / half_life_seconds)

# Score halves every 6 hours
HALF_LIFE = 6 * 3600
decayed = halflife_decay(100.0, 3600, HALF_LIFE)  # after 1 hour = 89.1
```

## Summary

Decaying counters in Redis combine hash storage for per-item score and timestamp with sorted sets for ranked queries. Scores are decayed on write by multiplying the existing score by an exponential factor based on elapsed time. A periodic background job prunes stale entries, keeping the leaderboard fresh without unbounded growth.
