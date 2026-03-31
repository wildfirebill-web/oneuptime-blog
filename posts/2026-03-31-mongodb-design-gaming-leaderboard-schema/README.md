# How to Design a Gaming Leaderboard Schema in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Design, Gaming, Leaderboard, Index

Description: Learn how to design a high-performance gaming leaderboard schema in MongoDB with real-time ranking, score updates, and efficient range queries.

---

## Why MongoDB for Gaming Leaderboards

Gaming leaderboards demand low-latency reads for rank lookups, high-throughput writes for score updates, and flexible query patterns for daily, weekly, and all-time boards. MongoDB's flexible document model, combined with powerful aggregation and indexing, makes it a strong choice for these requirements.

## Player Score Document

The core of a leaderboard is the player score document. Each player has one document per leaderboard type (global, regional, per-game-mode) which is updated in place:

```javascript
{
  _id: ObjectId("..."),
  playerId: "player-7842",
  displayName: "DragonSlayer99",
  gameMode: "ranked",
  region: "us-east",
  score: 18450,
  rank: 0,              // recomputed periodically or via aggregation
  wins: 142,
  losses: 38,
  winRate: 0.789,
  metadata: {
    avatarUrl: "https://cdn.example.com/avatars/7842.png",
    level: 45,
    clan: "Phoenix"
  },
  lastUpdated: ISODate("2024-03-15T12:00:00Z"),
  createdAt: ISODate("2023-06-01T00:00:00Z")
}
```

## Score Update Pattern

Use `$max` to ensure scores only ever increase (common in arcade-style games) or `$inc` for cumulative scoring:

```javascript
// Update score only if the new score is higher
await leaderboard.updateOne(
  { playerId: "player-7842", gameMode: "ranked", region: "us-east" },
  {
    $max: { score: 19200 },
    $inc: { wins: 1 },
    $set: {
      winRate: 143 / 181,
      lastUpdated: new Date()
    }
  },
  { upsert: true }
);
```

Using `upsert: true` means the document is created on first score submission, keeping the write path simple.

## Indexing for Rank Queries

The primary query pattern is "top N players globally" and "rank of a specific player":

```javascript
// Descending score index for top-N queries
db.leaderboard.createIndex({ gameMode: 1, score: -1 });

// Region-specific leaderboard
db.leaderboard.createIndex({ gameMode: 1, region: 1, score: -1 });

// Look up a single player's rank position
db.leaderboard.createIndex({ playerId: 1, gameMode: 1 }, { unique: true });
```

## Fetching the Top 100

```javascript
const top100 = await db.leaderboard.find(
  { gameMode: "ranked" },
  { projection: { displayName: 1, score: 1, wins: 1, winRate: 1 } }
)
.sort({ score: -1 })
.limit(100)
.toArray();
```

## Computing a Player's Rank

To find the exact rank of a player, count how many players have a higher score:

```javascript
async function getPlayerRank(playerId, gameMode) {
  const player = await db.leaderboard.findOne({ playerId, gameMode });
  if (!player) return null;

  const rank = await db.leaderboard.countDocuments({
    gameMode,
    score: { $gt: player.score }
  });

  return { rank: rank + 1, score: player.score };
}
```

This approach is accurate but becomes expensive at scale. For large leaderboards (millions of players) consider caching ranks in a scheduled job.

## Time-Based Leaderboard Boards

For daily and weekly leaderboards, use a separate collection with a TTL index:

```javascript
{
  _id: ObjectId("..."),
  playerId: "player-7842",
  gameMode: "ranked",
  period: "2024-W11",     // ISO week or YYYY-MM-DD for daily
  score: 4200,
  updatedAt: ISODate("2024-03-15T12:00:00Z")
}

// TTL index - documents expire after 8 days (keeps 7 full days of data)
db.leaderboard_weekly.createIndex(
  { updatedAt: 1 },
  { expireAfterSeconds: 691200 }
);

db.leaderboard_weekly.createIndex({ gameMode: 1, period: 1, score: -1 });
```

## Aggregation for Statistics

Use the aggregation pipeline to compute clan or team rankings:

```javascript
db.leaderboard.aggregate([
  { $match: { gameMode: "ranked" } },
  {
    $group: {
      _id: "$metadata.clan",
      totalScore: { $sum: "$score" },
      memberCount: { $sum: 1 },
      avgScore: { $avg: "$score" },
      topPlayer: { $max: "$score" }
    }
  },
  { $sort: { totalScore: -1 } },
  { $limit: 20 }
]);
```

## Pagination with Cursors

Avoid skip-based pagination for large leaderboards. Use range-based pagination with the last seen score as a cursor:

```javascript
// First page
const page1 = await db.leaderboard.find({ gameMode: "ranked" })
  .sort({ score: -1 })
  .limit(25)
  .toArray();

// Next page using last score as cursor
const lastScore = page1[page1.length - 1].score;
const page2 = await db.leaderboard.find({
  gameMode: "ranked",
  score: { $lt: lastScore }
})
.sort({ score: -1 })
.limit(25)
.toArray();
```

## Summary

A MongoDB gaming leaderboard schema centers on a per-player score document with compound indexes on game mode and score for efficient top-N queries. Score updates use `$max` or `$inc` with upsert for idempotent writes, while time-based boards use separate collections with TTL indexes for automatic expiry. For very large player bases, precomputing ranks on a schedule and caching them avoids expensive `countDocuments` calls at query time.
