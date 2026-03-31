# How to Build a Location-Based Social Feed with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Geospatial, Social Feed, Location, Real-Time

Description: Build a location-based social feed with Redis that surfaces nearby posts and events relevant to a user's current location.

---

Location-based feeds show content relevant to where you are right now - nearby restaurant reviews, local event posts, neighborhood discussions. Redis Geo indexes posts by location and returns them sorted by proximity in milliseconds.

## Storing Location-Tagged Posts

```python
import redis
import time
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def publish_post(post_id: str, user_id: str, lat: float, lon: float,
                 content: str, category: str = "general"):
    now = time.time()
    pipe = r.pipeline()
    # Add to geo index
    pipe.geoadd("posts:geo", [lon, lat, post_id])
    pipe.geoadd(f"posts:geo:{category}", [lon, lat, post_id])
    # Store post data
    pipe.hset(f"post:{post_id}", mapping={
        "user_id": user_id,
        "content": content,
        "lat": lat, "lon": lon,
        "category": category,
        "created_at": now,
        "likes": 0,
    })
    # Global feed (latest posts by time)
    pipe.zadd("posts:recent", {post_id: now})
    pipe.expire(f"post:{post_id}", 86400 * 30)  # Keep posts 30 days
    pipe.execute()
```

## Fetching Nearby Feed

```python
def get_nearby_feed(user_lat: float, user_lon: float,
                    radius_km: float = 5, limit: int = 20,
                    category: str = None) -> list:
    geo_key = f"posts:geo:{category}" if category else "posts:geo"
    results = r.geosearch(
        geo_key,
        longitude=user_lon, latitude=user_lat,
        radius=radius_km, unit="km",
        sort="ASC", count=limit,
        withdist=True,
    )

    if not results:
        return []

    pipe = r.pipeline()
    for post_id, _ in results:
        pipe.hgetall(f"post:{post_id}")
    post_data = pipe.execute()

    feed = []
    for (post_id, dist), data in zip(results, post_data):
        if data:
            feed.append({
                "post_id": post_id,
                "distance_km": round(float(dist), 2),
                **data,
            })
    return feed
```

## Trending Nearby

Combine geo proximity with recent engagement:

```python
def get_trending_nearby(user_lat: float, user_lon: float,
                        radius_km: float = 10) -> list:
    # Get posts in range
    results = r.geosearch(
        "posts:geo",
        longitude=user_lon, latitude=user_lat,
        radius=radius_km, unit="km",
        count=100,
    )

    if not results:
        return []

    # Rank by engagement score
    pipe = r.pipeline()
    for post_id in results:
        pipe.hget(f"post:{post_id}", "likes")
    likes = pipe.execute()

    scored = sorted(
        zip(results, likes),
        key=lambda x: int(x[1] or 0),
        reverse=True,
    )
    return [{"post_id": pid, "likes": int(l or 0)} for pid, l in scored[:20]]
```

## Liking a Post

```python
def like_post(post_id: str, user_id: str):
    like_key = f"post:{post_id}:likes"
    if r.sadd(like_key, user_id):  # Returns 1 if new
        r.hincrby(f"post:{post_id}", "likes", 1)
        r.expire(like_key, 86400 * 30)
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your location feed API - high response times indicate the geo index is growing too large and may need partitioning.

```bash
redis-cli ZCARD posts:geo
```

## Summary

Redis Geo indexes posts by coordinate, enabling proximity-sorted feed queries with GEOSEARCH. Per-category geo keys allow filtered feeds (events, restaurants, discussions) without client-side filtering overhead. Combining geo proximity with engagement score sorting creates a "relevant nearby" feed that balances freshness and popularity.
