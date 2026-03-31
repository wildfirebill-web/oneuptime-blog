# How to Use $rank for Competition Ranking in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Window Function, Ranking, Aggregation, Analytics

Description: Learn how to use the $rank window operator in MongoDB to assign competition rankings with gaps for tied values in leaderboards and analytics queries.

---

## What Is $rank in MongoDB

`$rank` is a window function operator available in `$setWindowFields` (MongoDB 5.0+). It assigns a rank to each document within a partition based on the sort order. When two documents have equal sort values, they receive the same rank, and the next rank is incremented by the number of tied documents (like competition ranking - 1, 2, 2, 4).

## Basic Syntax

```javascript
{
  $setWindowFields: {
    partitionBy: "$partitionField",  // optional grouping
    sortBy: { scoreField: -1 },       // determines rank order
    output: {
      rank: { $rank: {} }             // no parameters needed
    }
  }
}
```

## Setup - Leaderboard Dataset

```javascript
db.leaderboard.insertMany([
  { player: "Alice",  game: "Chess",  score: 2800, region: "NA" },
  { player: "Bob",    game: "Chess",  score: 2750, region: "NA" },
  { player: "Carol",  game: "Chess",  score: 2750, region: "EU" },
  { player: "Dave",   game: "Chess",  score: 2700, region: "NA" },
  { player: "Eve",    game: "Chess",  score: 2650, region: "EU" },
  { player: "Frank",  game: "Chess",  score: 2600, region: "NA" },
  { player: "Grace",  game: "Go",     score: 3100, region: "AS" },
  { player: "Henry",  game: "Go",     score: 3050, region: "NA" },
  { player: "Iris",   game: "Go",     score: 3050, region: "AS" },
  { player: "Jack",   game: "Go",     score: 2950, region: "EU" }
]);
```

## Example 1 - Global Ranking per Game

Rank players within each game by score descending:

```javascript
db.leaderboard.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$game",
      sortBy: { score: -1 },
      output: {
        gameRank: { $rank: {} }
      }
    }
  },
  { $sort: { game: 1, gameRank: 1 } }
]);
```

```text
{ player: "Alice", game: "Chess", score: 2800, gameRank: 1 }
{ player: "Bob",   game: "Chess", score: 2750, gameRank: 2 }
{ player: "Carol", game: "Chess", score: 2750, gameRank: 2 }  <- tied at 2
{ player: "Dave",  game: "Chess", score: 2700, gameRank: 4 }  <- gap to 4
{ player: "Eve",   game: "Chess", score: 2650, gameRank: 5 }
```

## Example 2 - Ranking Within Region per Game

Partition by both game and region to get region-specific leaderboards:

```javascript
db.leaderboard.aggregate([
  {
    $setWindowFields: {
      partitionBy: { game: "$game", region: "$region" },
      sortBy: { score: -1 },
      output: {
        regionalRank: { $rank: {} }
      }
    }
  },
  {
    $setWindowFields: {
      partitionBy: "$game",
      sortBy: { score: -1 },
      output: {
        globalRank: { $rank: {} }
      }
    }
  },
  { $sort: { game: 1, globalRank: 1 } }
]);
```

## Example 3 - Top N per Partition

Find only the top 3 players in each game:

```javascript
db.leaderboard.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$game",
      sortBy: { score: -1 },
      output: {
        gameRank: { $rank: {} }
      }
    }
  },
  { $match: { gameRank: { $lte: 3 } } },
  { $sort: { game: 1, gameRank: 1 } }
]);
```

## Example 4 - Ranking with Multiple Sort Keys

Break ties using a secondary sort field:

```javascript
// Add tiebreaker: higher wins count breaks score ties
db.leaderboard.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$game",
      sortBy: { score: -1, wins: -1 },  // primary score, secondary wins
      output: {
        rank: { $rank: {} }
      }
    }
  }
]);
```

## Example 5 - Rank Change Over Time

Compare rankings between two time periods:

```javascript
// Assuming two periods of data with a "period" field
db.leaderboard.aggregate([
  {
    $setWindowFields: {
      partitionBy: { game: "$game", period: "$period" },
      sortBy: { score: -1 },
      output: {
        periodRank: { $rank: {} }
      }
    }
  },
  {
    $group: {
      _id: { game: "$game", player: "$player" },
      ranks: { $push: { period: "$period", rank: "$periodRank" } }
    }
  },
  {
    $addFields: {
      currentRank: {
        $let: {
          vars: {
            current: {
              $filter: {
                input: "$ranks",
                cond: { $eq: ["$$this.period", "current"] }
              }
            }
          },
          in: { $arrayElemAt: ["$$current.rank", 0] }
        }
      },
      previousRank: {
        $let: {
          vars: {
            prev: {
              $filter: {
                input: "$ranks",
                cond: { $eq: ["$$this.period", "previous"] }
              }
            }
          },
          in: { $arrayElemAt: ["$$prev.rank", 0] }
        }
      }
    }
  },
  {
    $addFields: {
      rankChange: { $subtract: ["$previousRank", "$currentRank"] }
    }
  }
]);
```

## Example 6 - Percentile Based on Rank

Compute what percentile each rank represents:

```javascript
db.leaderboard.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$game",
      sortBy: { score: -1 },
      output: {
        rank: { $rank: {} },
        totalPlayers: {
          $count: {},
          window: { documents: ["unbounded", "unbounded"] }
        }
      }
    }
  },
  {
    $addFields: {
      percentile: {
        $multiply: [
          { $divide: [{ $subtract: ["$totalPlayers", "$rank"] }, "$totalPlayers"] },
          100
        ]
      }
    }
  }
]);
```

## $rank vs $denseRank

```text
Scores: 100, 90, 90, 80

$rank:      1, 2, 2, 4   (gap after ties)
$denseRank: 1, 2, 2, 3   (no gap - consecutive)

Use $rank for:
- Competition rankings (Olympic medals)
- Cases where gaps indicate "missing" positions
- Standard SQL-style ranking behavior

Use $denseRank for:
- Tier assignments (Bronze/Silver/Gold)
- Cases where gaps are confusing
- Grouping tied values into same tier
```

## Summary

`$rank` in MongoDB's `$setWindowFields` implements competition-style ranking where tied values share the same rank and the next rank is skipped by the number of ties. Use it for leaderboards, tournament standings, and any scenario where gaps in ranking convey meaning. Combine it with `$match` to find top-N results per partition, with multiple sort keys for deterministic tiebreaking, and with `$count` to derive percentile standings alongside raw ranks.
