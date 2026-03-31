# How to Build a Content Recommendation Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Recommendation, Cache

Description: Cache personalized content recommendations in Redis to serve instant results while running expensive ML recommendation models in the background.

---

Recommendation models are expensive to run - they involve collaborative filtering, embedding lookups, and ranking passes. But users expect personalized recommendations to load instantly. The solution is to pre-compute recommendations in the background and cache them in Redis so every page load is a fast cache read.

## Architecture

1. A background job runs the recommendation model and writes results to Redis.
2. The API reads recommendations directly from Redis (sub-millisecond).
3. If a user has no cached recommendations (new user, cache expired), serve popular/trending content as fallback.
4. Personalization refreshes every 1-4 hours depending on user activity.

## Setup

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

RECO_PREFIX = "reco"
TRENDING_KEY = "reco:trending"
POPULAR_KEY = "reco:popular"
RECO_TTL = 3600          # 1 hour for personalized
TRENDING_TTL = 900       # 15 minutes for trending
```

## Writing Recommendations (Background Job)

```python
def store_recommendations(user_id: str, content_ids: list[str], model_version: str, scores: list[float] = None):
    key = f"{RECO_PREFIX}:user:{user_id}"
    reco_data = {
        "content_ids": content_ids,
        "model_version": model_version,
        "generated_at": int(time.time()),
        "scores": scores or [1.0] * len(content_ids)
    }
    r.setex(key, RECO_TTL, json.dumps(reco_data))

def store_trending(content_ids: list[str], context: str = "global"):
    key = f"{TRENDING_KEY}:{context}"
    r.setex(key, TRENDING_TTL, json.dumps({
        "content_ids": content_ids,
        "generated_at": int(time.time()),
        "context": context
    }))
```

## Reading Recommendations

```python
def get_recommendations(user_id: str, limit: int = 20, context: str = "home") -> dict:
    user_key = f"{RECO_PREFIX}:user:{user_id}"
    cached = r.get(user_key)

    if cached:
        reco = json.loads(cached)
        content_ids = reco["content_ids"][:limit]
        return {
            "source": "personalized",
            "model_version": reco["model_version"],
            "content_ids": content_ids,
            "generated_at": reco["generated_at"]
        }

    # Fallback to trending
    return get_trending(context, limit)

def get_trending(context: str = "global", limit: int = 20) -> dict:
    key = f"{TRENDING_KEY}:{context}"
    cached = r.get(key)

    if cached:
        data = json.loads(cached)
        return {
            "source": "trending",
            "content_ids": data["content_ids"][:limit],
            "generated_at": data["generated_at"]
        }

    # Final fallback: popular all-time
    popular = r.lrange(POPULAR_KEY, 0, limit - 1)
    return {
        "source": "popular",
        "content_ids": popular,
        "generated_at": int(time.time())
    }
```

## Context-Aware Recommendations

Different surfaces (home, post-play, search) can have separate recommendation lists:

```python
def get_similar_content(content_id: str, limit: int = 10) -> list:
    key = f"{RECO_PREFIX}:similar:{content_id}"
    cached = r.get(key)
    if cached:
        data = json.loads(cached)
        return data["content_ids"][:limit]
    return []

def store_similar_content(content_id: str, similar_ids: list[str]):
    key = f"{RECO_PREFIX}:similar:{content_id}"
    r.setex(key, 86400, json.dumps({  # 24 hours - similar content rarely changes
        "content_ids": similar_ids,
        "generated_at": int(time.time())
    }))
```

## Batch Recommendation Serving

For feeds that show multiple recommendation modules:

```python
def get_homepage_feed(user_id: str) -> dict:
    pipe = r.pipeline()
    pipe.get(f"{RECO_PREFIX}:user:{user_id}")
    pipe.get(f"{TRENDING_KEY}:global")
    pipe.get(f"{RECO_PREFIX}:user:{user_id}:continue_watching")
    results = pipe.execute()

    personalized_raw, trending_raw, continue_raw = results

    return {
        "for_you": json.loads(personalized_raw)["content_ids"][:10] if personalized_raw else [],
        "trending": json.loads(trending_raw)["content_ids"][:10] if trending_raw else [],
        "continue_watching": json.loads(continue_raw)["content_ids"][:5] if continue_raw else []
    }
```

## Cache Warming on User Login

```python
def warm_user_cache_if_needed(user_id: str):
    key = f"{RECO_PREFIX}:user:{user_id}"
    ttl = r.ttl(key)
    if ttl < 600:  # Less than 10 minutes remaining - trigger refresh
        r.publish("reco:refresh_requests", json.dumps({
            "user_id": user_id,
            "priority": "high",
            "ts": int(time.time())
        }))
```

## Summary

A Redis recommendation cache separates the expensive model execution (background job) from the fast read path (Redis get). Personalized recommendations are stored per user with hourly TTLs, similar-content lists are cached with 24-hour TTLs, and trending content refreshes every 15 minutes. New and logged-out users fall through to trending or popular content without any model call.
