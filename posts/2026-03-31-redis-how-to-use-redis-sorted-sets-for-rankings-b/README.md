# How to Use Redis Sorted Sets for Rankings (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sorted Sets, Leaderboard, Rankings, Beginner, Node.js, Python

Description: A beginner-friendly guide to using Redis Sorted Sets to build leaderboards and rankings, with real-world examples for games, content, and user activity.

---

## What Are Redis Sorted Sets?

A Sorted Set is a collection of unique members, each with a numeric score. Redis keeps them automatically sorted by score. You can retrieve members by rank, by score range, or look up any member's rank instantly.

```bash
# Add members with scores
ZADD leaderboard 9500 "alice"
ZADD leaderboard 8800 "bob"
ZADD leaderboard 9100 "carol"

# Get top 3 (highest scores first)
ZREVRANGE leaderboard 0 2 WITHSCORES
# 1) "alice"
# 2) "9500"
# 3) "carol"
# 4) "9100"
# 5) "bob"
# 6) "8800"
```

## Adding and Updating Scores

```bash
# Add a new player
ZADD leaderboard 7200 "dave"

# Update an existing player's score (ZADD updates if member exists)
ZADD leaderboard 9700 "alice"  # alice now has 9700

# Increment a player's score by 100 (atomic - no read needed)
ZINCRBY leaderboard 100 "bob"  # bob now has 8900
```

## Getting Rankings

```bash
# Get rank of a player (0-based, 0 = lowest score)
ZRANK leaderboard "alice"      # Returns position from bottom

# Get rank from top (0 = highest score)
ZREVRANK leaderboard "alice"   # Returns position from top

# Get a player's score
ZSCORE leaderboard "alice"     # Returns "9700"

# Get top 10 players with scores (highest first)
ZREVRANGE leaderboard 0 9 WITHSCORES

# Get bottom 10 players (lowest first)
ZRANGE leaderboard 0 9 WITHSCORES
```

## Node.js Leaderboard Example

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

const LEADERBOARD_KEY = 'game:leaderboard:global';

async function submitScore(userId, score) {
  // ZADD overwrites if user already exists with a higher score
  await redis.zadd(LEADERBOARD_KEY, score, `user:${userId}`);
  console.log(`Score ${score} submitted for user ${userId}`);
}

async function getTopPlayers(n = 10) {
  const results = await redis.zrevrange(LEADERBOARD_KEY, 0, n - 1, 'WITHSCORES');

  const players = [];
  for (let i = 0; i < results.length; i += 2) {
    const userId = results[i].replace('user:', '');
    const score = parseInt(results[i + 1]);
    const rank = i / 2 + 1;
    players.push({ rank, userId, score });
  }

  return players;
}

async function getPlayerRank(userId) {
  const rank = await redis.zrevrank(LEADERBOARD_KEY, `user:${userId}`);
  const score = await redis.zscore(LEADERBOARD_KEY, `user:${userId}`);

  if (rank === null) return null;

  return {
    userId,
    rank: rank + 1, // Convert 0-based to 1-based
    score: parseFloat(score)
  };
}

async function getPlayersAroundRank(userId, range = 2) {
  const rank = await redis.zrevrank(LEADERBOARD_KEY, `user:${userId}`);
  if (rank === null) return [];

  const start = Math.max(0, rank - range);
  const end = rank + range;

  const results = await redis.zrevrange(LEADERBOARD_KEY, start, end, 'WITHSCORES');

  const players = [];
  for (let i = 0; i < results.length; i += 2) {
    players.push({
      rank: start + i / 2 + 1,
      userId: results[i].replace('user:', ''),
      score: parseInt(results[i + 1]),
      isCurrentUser: results[i] === `user:${userId}`
    });
  }

  return players;
}

// Usage
await submitScore(1, 9500);
await submitScore(2, 8800);
await submitScore(3, 9100);

const top10 = await getTopPlayers(10);
console.log('Top 10:', top10);

const myRank = await getPlayerRank(1);
console.log('My rank:', myRank);
```

## Python Leaderboard Example

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def submit_score(user_id: int, score: float, board: str = 'global'):
    r.zadd(f"leaderboard:{board}", {f"user:{user_id}": score})

def get_top_players(board: str = 'global', n: int = 10) -> list:
    results = r.zrevrange(f"leaderboard:{board}", 0, n - 1, withscores=True)
    return [
        {'rank': i + 1, 'userId': member.replace('user:', ''), 'score': score}
        for i, (member, score) in enumerate(results)
    ]

def get_player_rank(user_id: int, board: str = 'global') -> dict | None:
    key = f"leaderboard:{board}"
    rank = r.zrevrank(key, f"user:{user_id}")
    score = r.zscore(key, f"user:{user_id}")

    if rank is None:
        return None

    return {'userId': user_id, 'rank': rank + 1, 'score': score}

# Usage
submit_score(1, 9500)
submit_score(2, 8800)
submit_score(3, 9100)

print(get_top_players())
print(get_player_rank(1))
```

## Range Queries by Score

```bash
# Get players with scores between 8000 and 9000
ZRANGEBYSCORE leaderboard 8000 9000 WITHSCORES

# Get players above 9000 (exclusive)
ZRANGEBYSCORE leaderboard (9000 +inf WITHSCORES

# Count players with scores in range
ZCOUNT leaderboard 8000 9500
```

## Time-Based Leaderboards

Use timestamps as scores for "most recent" rankings:

```bash
# Add an event with timestamp as score
ZADD events:recent 1711900000 "event:login:user-42"
ZADD events:recent 1711900100 "event:purchase:user-15"

# Get the 10 most recent events
ZREVRANGE events:recent 0 9 WITHSCORES

# Get events in the last hour
ZRANGEBYSCORE events:recent 1711896400 +inf
```

## Weekly Leaderboard Reset

```javascript
async function getWeeklyLeaderboard() {
  const weekKey = getWeekKey(); // e.g., "leaderboard:week:2026-13"
  return redis.zrevrange(weekKey, 0, 9, 'WITHSCORES');
}

async function submitWeeklyScore(userId, score) {
  const weekKey = getWeekKey();
  await redis.zadd(weekKey, score, `user:${userId}`);
  await redis.expire(weekKey, 604800); // Expire after 1 week
}

function getWeekKey() {
  const now = new Date();
  const week = Math.ceil(now.getDate() / 7);
  return `leaderboard:week:${now.getFullYear()}-${week}`;
}
```

## Summary

Redis Sorted Sets are purpose-built for leaderboards and rankings. ZADD adds members with scores, ZREVRANK gives any member's position in O(log N) time, and ZREVRANGE retrieves the top N players efficiently. Use ZINCRBY for atomic score increments without race conditions. Sorted Sets handle millions of players with sub-millisecond queries, making them ideal for real-time gaming leaderboards and content rankings.
