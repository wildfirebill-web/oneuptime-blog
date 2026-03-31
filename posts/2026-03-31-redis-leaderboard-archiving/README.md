# How to Implement Leaderboard Archiving with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Leaderboard, Archiving, History, Sorted Set

Description: Implement leaderboard archiving in Redis to preserve historical rankings before resets and enable season-over-season comparisons.

---

Leaderboard resets are exciting for players - but you need to archive the final standings before wiping them. Redis makes it straightforward to snapshot Sorted Sets into archive keys with a TTL or permanent storage.

## Archiving a Leaderboard Snapshot

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def archive_leaderboard(source_key: str, archive_name: str,
                        top_n: int = None, ttl_days: int = 365):
    """Copy a leaderboard into an archive key before reset."""
    archive_key = f"archive:{archive_name}"

    if top_n is not None:
        entries = r.zrevrange(source_key, 0, top_n - 1, withscores=True)
    else:
        entries = r.zrange(source_key, 0, -1, withscores=True)

    if not entries:
        return 0

    pipe = r.pipeline()
    mapping = {player: score for player, score in entries}
    pipe.zadd(archive_key, mapping)
    pipe.expire(archive_key, ttl_days * 86400)
    # Store archive metadata
    pipe.hset(f"archive:meta:{archive_name}", mapping={
        "archived_at": time.time(),
        "player_count": len(entries),
        "source_key": source_key,
    })
    pipe.execute()
    return len(entries)
```

## Season-Based Archiving

```python
def close_season(season_id: str, leaderboard_key: str):
    archive_name = f"season:{season_id}"
    count = archive_leaderboard(leaderboard_key, archive_name, top_n=1000)

    # Reset the live leaderboard
    r.delete(leaderboard_key)

    return {
        "season": season_id,
        "archived_players": count,
        "archive_key": f"archive:{archive_name}",
    }
```

## Retrieving Historical Rankings

```python
def get_archive_top(archive_name: str, n: int = 10) -> list:
    archive_key = f"archive:{archive_name}"
    entries = r.zrevrange(archive_key, 0, n - 1, withscores=True)
    return [
        {"rank": i + 1, "player": pid, "score": score}
        for i, (pid, score) in enumerate(entries)
    ]

def get_player_archive_rank(archive_name: str, player_id: str) -> dict:
    archive_key = f"archive:{archive_name}"
    rank = r.zrevrank(archive_key, player_id)
    score = r.zscore(archive_key, player_id)
    return {
        "archive": archive_name,
        "rank": (rank + 1) if rank is not None else None,
        "score": score,
    }
```

## Listing Available Archives

```python
def list_archives() -> list:
    archive_meta_keys = r.keys("archive:meta:*")
    archives = []
    for key in archive_meta_keys:
        meta = r.hgetall(key)
        name = key.replace("archive:meta:", "")
        archives.append({"name": name, **meta})
    return sorted(archives, key=lambda x: float(x.get("archived_at", 0)), reverse=True)
```

## Comparing Seasons

```python
def compare_seasons(player_id: str, season_a: str, season_b: str) -> dict:
    rank_a = get_player_archive_rank(f"season:{season_a}", player_id)
    rank_b = get_player_archive_rank(f"season:{season_b}", player_id)
    improvement = None
    if rank_a["rank"] and rank_b["rank"]:
        improvement = rank_b["rank"] - rank_a["rank"]  # negative = improved
    return {
        "player": player_id,
        season_a: rank_a,
        season_b: rank_b,
        "rank_change": improvement,
    }
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your archiving job and receive alerts if a season close fails - you do not want to reset a leaderboard without a successful archive.

```bash
redis-cli KEYS "archive:season:*" | wc -l
```

## Summary

Leaderboard archiving copies a live Sorted Set into a dated archive key before reset. A long TTL (1 year) keeps historical data without permanent storage concerns. Per-archive metadata Hashes store timestamps and player counts for audit trails. Season comparison queries enable player progression features and retention campaigns.
