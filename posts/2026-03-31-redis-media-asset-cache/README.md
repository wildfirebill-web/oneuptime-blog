# How to Build a Media Asset Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Media, Cache

Description: Cache media asset metadata, CDN URLs, and processing status in Redis to eliminate repeated lookups and deliver instant asset resolution for video and image services.

---

Media platforms serve millions of asset lookups daily: video metadata, thumbnail URLs, subtitle tracks, and processing status. Fetching this from a database on every player load creates unnecessary load. Redis caches asset metadata so player initialization happens in a single fast lookup.

## What to Cache

- Video metadata (title, duration, codec, bitrate variants)
- CDN URLs for each quality tier
- Thumbnail and poster image URLs
- Processing/transcoding status
- Subtitle/caption track locations

## Setup

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

ASSET_PREFIX = "media:asset"
PROCESSING_PREFIX = "media:processing"
THUMBNAIL_PREFIX = "media:thumb"
CDN_BASE = "https://cdn.example.com"

ASSET_TTL = 3600         # 1 hour for video metadata
PROCESSING_TTL = 86400   # 24 hours for processing status
THUMB_TTL = 86400        # 24 hours for thumbnail URLs
```

## Caching Video Asset Metadata

```python
def cache_asset(asset_id: str, metadata: dict):
    key = f"{ASSET_PREFIX}:{asset_id}"
    r.setex(key, ASSET_TTL, json.dumps({
        **metadata,
        "cached_at": int(time.time())
    }))

def get_asset(asset_id: str) -> dict | None:
    key = f"{ASSET_PREFIX}:{asset_id}"
    cached = r.get(key)
    if cached:
        return json.loads(cached)

    # Cache miss - fetch from database
    metadata = fetch_asset_from_db(asset_id)
    if metadata:
        cache_asset(asset_id, metadata)
    return metadata

def fetch_asset_from_db(asset_id: str) -> dict | None:
    # Replace with actual DB query
    return {
        "id": asset_id,
        "title": "Big Buck Bunny",
        "duration_seconds": 596,
        "status": "ready",
        "variants": [
            {"quality": "1080p", "bitrate_kbps": 8000, "url": f"{CDN_BASE}/{asset_id}/1080p.m3u8"},
            {"quality": "720p", "bitrate_kbps": 4000, "url": f"{CDN_BASE}/{asset_id}/720p.m3u8"},
            {"quality": "480p", "bitrate_kbps": 2000, "url": f"{CDN_BASE}/{asset_id}/480p.m3u8"},
            {"quality": "360p", "bitrate_kbps": 1000, "url": f"{CDN_BASE}/{asset_id}/360p.m3u8"}
        ],
        "subtitles": [
            {"lang": "en", "url": f"{CDN_BASE}/{asset_id}/en.vtt"},
            {"lang": "es", "url": f"{CDN_BASE}/{asset_id}/es.vtt"}
        ],
        "poster": f"{CDN_BASE}/{asset_id}/poster.jpg"
    }
```

## Bulk Asset Lookup for Playlists

```python
def get_assets_bulk(asset_ids: list[str]) -> dict:
    keys = [f"{ASSET_PREFIX}:{aid}" for aid in asset_ids]
    cached_values = r.mget(keys)

    results = {}
    miss_ids = []

    for aid, raw in zip(asset_ids, cached_values):
        if raw:
            results[aid] = json.loads(raw)
        else:
            miss_ids.append(aid)

    if miss_ids:
        pipe = r.pipeline()
        for aid in miss_ids:
            metadata = fetch_asset_from_db(aid)
            if metadata:
                results[aid] = metadata
                pipe.setex(f"{ASSET_PREFIX}:{aid}", ASSET_TTL, json.dumps(metadata))
        pipe.execute()

    return results
```

## Processing Status Tracking

```python
def set_processing_status(asset_id: str, status: str, progress_pct: int = 0, error: str = None):
    key = f"{PROCESSING_PREFIX}:{asset_id}"
    data = {
        "asset_id": asset_id,
        "status": status,  # queued, processing, ready, failed
        "progress_pct": progress_pct,
        "updated_at": int(time.time())
    }
    if error:
        data["error"] = error

    r.setex(key, PROCESSING_TTL, json.dumps(data))

    # Publish processing update for real-time progress UI
    r.publish(f"media:processing:{asset_id}", json.dumps(data))

def get_processing_status(asset_id: str) -> dict | None:
    key = f"{PROCESSING_PREFIX}:{asset_id}"
    cached = r.get(key)
    return json.loads(cached) if cached else None
```

## Thumbnail URL Cache

```python
def get_thumbnail_url(asset_id: str, width: int = 640, time_offset: int = 0) -> str:
    cache_key = f"{THUMBNAIL_PREFIX}:{asset_id}:{width}:{time_offset}"
    cached = r.get(cache_key)

    if cached:
        return cached

    # Generate thumbnail URL (typically involves signed CDN URL)
    thumb_url = f"{CDN_BASE}/{asset_id}/thumbs/{time_offset:06d}_{width}w.jpg"
    r.setex(cache_key, THUMB_TTL, thumb_url)
    return thumb_url
```

## Cache Invalidation on Asset Update

```python
def invalidate_asset(asset_id: str):
    pipe = r.pipeline()
    pipe.delete(f"{ASSET_PREFIX}:{asset_id}")
    pipe.delete(f"{PROCESSING_PREFIX}:{asset_id}")

    # Delete all thumbnail cache entries for this asset
    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, match=f"{THUMBNAIL_PREFIX}:{asset_id}:*", count=100)
        if keys:
            pipe.delete(*keys)
        if cursor == 0:
            break

    pipe.execute()
```

## Summary

A Redis media asset cache stores video metadata, CDN URLs, and processing status with TTLs tuned to how frequently each type changes. Bulk pipeline reads power playlist loading in a single round trip, processing status is published via Pub/Sub for real-time progress UIs, and targeted invalidation clears all variants of an asset when its source file changes.
