# How to Design a Leaderboard Using Redis in a System Design Interview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, System Design, Leaderboard, Interview, Sorted Set

Description: How to design a scalable Redis leaderboard in a system design interview using sorted sets, with ranking, pagination, and real-time updates.

---

Designing a leaderboard is a popular system design interview question, and Redis sorted sets are the canonical answer. This post covers how to explain the design clearly, handle edge cases, and scale it.

## Define the Requirements

Start by clarifying:
- Global leaderboard or segmented (per-game, per-week)?
- How often are scores updated? (real-time writes)
- How many players? (millions or billions?)
- Do you need surroundings rank (show player's neighbors)?

## Core Data Structure: Sorted Set

A Redis sorted set stores members ranked by a floating-point score. All ranking operations are O(log N):

```bash
# Add or update player scores
ZADD game:leaderboard 9500 "player:alice"
ZADD game:leaderboard 8200 "player:bob"
ZADD game:leaderboard 11000 "player:carol"

# Increment score (atomic)
ZINCRBY game:leaderboard 500 "player:alice"

# Top 10 players (highest score first)
ZREVRANGE game:leaderboard 0 9 WITHSCORES

# Player's rank (0-indexed, 0 = highest)
ZREVRANK game:leaderboard "player:alice"

# Player's score
ZSCORE game:leaderboard "player:alice"
```

## Pagination

For browsing pages of the leaderboard:

```bash
# Page 1: ranks 0-19
ZREVRANGE game:leaderboard 0 19 WITHSCORES

# Page 2: ranks 20-39
ZREVRANGE game:leaderboard 20 39 WITHSCORES
```

## Show a Player's Surroundings

A common UX need - show the 5 players above and below the current player:

```bash
# Get player rank
rank = ZREVRANK game:leaderboard "player:alice"

# Get surrounding players
start = max(0, rank - 5)
ZREVRANGE game:leaderboard {start} {rank + 5} WITHSCORES
```

## Segmented Leaderboards

For weekly or per-game leaderboards, use separate keys:

```bash
ZADD game:leaderboard:week:2026-w13 500 "player:alice"
ZADD game:leaderboard:game:chess 1200 "player:alice"
```

Set TTL on time-boxed leaderboards:

```bash
EXPIRE game:leaderboard:week:2026-w13 604800  # 7 days
```

## Scaling Considerations

For billions of players:
- Partition by user ID range or region into multiple sorted sets
- Merge top-N from each partition for global top rankings
- Consider approximate ranking using sketches for huge scale

For write-heavy games:
- Buffer score updates in application layer, flush to Redis in batches
- Use `ZINCRBY` for atomic increments rather than read-modify-write

## What to Say in an Interview

1. "I'll use a Redis sorted set, which gives O(log N) rank queries..."
2. "Scores are updated with ZINCRBY for atomic increment..."
3. "Top-N is ZREVRANGE 0 N-1, player rank is ZREVRANK..."
4. "For weekly boards, I use separate keys with TTL..."
5. "At massive scale, I'd partition by user ID and merge top results..."

## Summary

Redis sorted sets make leaderboard design elegant - O(log N) inserts, updates, and rank queries. In an interview, show you can handle pagination, surroundings rank, segmented boards, and know when to think about partitioning for extreme scale. Atomic ZINCRBY is key for concurrent score updates.
