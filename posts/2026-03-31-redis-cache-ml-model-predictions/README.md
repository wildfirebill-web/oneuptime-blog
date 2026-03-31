# How to Cache ML Model Predictions with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Machine Learning, Cache, Inference, Performance

Description: Cache ML model prediction results in Redis to reduce inference latency and GPU costs when the same inputs are requested repeatedly in production.

---

Model inference is expensive - a typical recommendation or fraud detection model takes 10-100ms per request. When the same inputs appear repeatedly (popular products, known users), caching predictions in Redis can serve 90%+ of requests in under 1ms.

## When Prediction Caching Makes Sense

- Recommendation models where the top-N results for a user change slowly
- Fraud scoring on recurring transactions with similar feature vectors
- Content classification where the same text is submitted many times
- Search ranking for popular queries

## Cache Key Design

The cache key must uniquely represent the model inputs:

```python
import hashlib
import json
import redis

r = redis.Redis(host="redis", port=6379, decode_responses=True)

def make_cache_key(model_name: str, model_version: str, inputs: dict) -> str:
    # Sort keys for deterministic hashing
    input_str = json.dumps(inputs, sort_keys=True)
    hash_val = hashlib.sha256(input_str.encode()).hexdigest()[:16]
    return f"prediction:{model_name}:{model_version}:{hash_val}"
```

## Read-Through Cache

```python
import time

def predict_with_cache(
    model, model_name: str, version: str,
    inputs: dict, ttl: int = 300
) -> dict:
    key = make_cache_key(model_name, version, inputs)

    # Try cache first
    cached = r.get(key)
    if cached:
        result = json.loads(cached)
        result["cache_hit"] = True
        return result

    # Cache miss: run inference
    start = time.time()
    prediction = model.predict(inputs)
    latency_ms = (time.time() - start) * 1000

    result = {
        "prediction": prediction,
        "latency_ms": round(latency_ms, 2),
        "cache_hit": False
    }

    # Store in cache
    r.setex(key, ttl, json.dumps(result))
    return result
```

## Caching Top-N Recommendations

For ranking models, cache the ordered list as a Redis List:

```python
def get_recommendations(user_id: int, model, top_n: int = 10) -> list:
    key = f"recs:{user_id}:top{top_n}"

    cached = r.lrange(key, 0, -1)
    if cached:
        return [int(x) for x in cached]

    # Compute recommendations
    recs = model.recommend(user_id, top_n=top_n)

    # Cache as list
    pipeline = r.pipeline()
    pipeline.delete(key)
    pipeline.rpush(key, *recs)
    pipeline.expire(key, 600)  # 10-minute TTL
    pipeline.execute()

    return recs
```

## Cache Invalidation on Model Update

When deploying a new model version, invalidate old predictions:

```python
def invalidate_model_cache(model_name: str, old_version: str):
    # Use SCAN to find and delete all keys for the old version
    cursor = 0
    pattern = f"prediction:{model_name}:{old_version}:*"
    deleted = 0

    while True:
        cursor, keys = r.scan(cursor, match=pattern, count=200)
        if keys:
            r.delete(*keys)
            deleted += len(keys)
        if cursor == 0:
            break

    print(f"Invalidated {deleted} cached predictions for {model_name} {old_version}")
```

## Monitoring Cache Effectiveness

```python
def track_cache_metrics(cache_hit: bool):
    if cache_hit:
        r.incr("metrics:prediction_cache_hits")
    else:
        r.incr("metrics:prediction_cache_misses")

def get_hit_rate() -> float:
    hits = int(r.get("metrics:prediction_cache_hits") or 0)
    misses = int(r.get("metrics:prediction_cache_misses") or 0)
    total = hits + misses
    return hits / total if total > 0 else 0.0
```

## Choosing TTL by Staleness Tolerance

| Model Type | Recommended TTL |
|---|---|
| Real-time fraud score | 60 seconds |
| User recommendations | 5-15 minutes |
| Search ranking | 30-60 minutes |
| Content category | 24 hours |

## Summary

Caching ML predictions in Redis reduces inference latency from tens of milliseconds to sub-millisecond for repeated inputs. Build a deterministic cache key from a sorted hash of model inputs and version, store results with appropriate TTLs, and use SCAN-based invalidation when deploying new model versions. Monitor hit rate to confirm the cache provides meaningful cost and latency savings.
