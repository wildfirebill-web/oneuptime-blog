# How to Use $merge to Upsert Aggregation Results in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $merge, Upsert, Pipeline Stage, NoSQL

Description: Learn how to use MongoDB's $merge stage to write aggregation results back to a collection, supporting insert, replace, update, and upsert behaviors.

---

## What Is the $merge Stage?

The `$merge` stage writes the output of an aggregation pipeline into a collection. Unlike `$out`, `$merge` can merge results into an existing collection rather than replacing it entirely. It supports multiple merge behaviors per document: insert, replace, update, or keep.

```javascript
{
  $merge: {
    into: "targetCollection",
    on: "matchField",           // field(s) to match existing documents
    whenMatched: "merge",       // behavior when match found
    whenNotMatched: "insert"    // behavior when no match found
  }
}
```

## whenMatched Options

| Value | Behavior |
|---|---|
| `"replace"` | Replace existing document with pipeline result |
| `"merge"` | Merge fields from pipeline result into existing document |
| `"keepExisting"` | Keep the existing document unchanged |
| `"fail"` | Throw an error if a match is found |
| `[pipeline]` | Apply an aggregation pipeline to merge |

## whenNotMatched Options

| Value | Behavior |
|---|---|
| `"insert"` | Insert the pipeline result as a new document |
| `"discard"` | Don't insert if no match exists |
| `"fail"` | Throw an error if no match is found |

## Basic Example - Incrementally Update Summary Collection

Compute daily order totals and merge them into a summary collection:

```javascript
db.orders.aggregate([
  {
    $match: {
      orderDate: {
        $gte: new Date("2024-01-01"),
        $lt: new Date("2024-01-02")
      }
    }
  },
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m-%d", date: "$orderDate" } },
      totalOrders: { $sum: 1 },
      totalRevenue: { $sum: "$amount" }
    }
  },
  {
    $merge: {
      into: "dailySummary",
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

Existing summaries for that date are replaced; new dates are inserted.

## Merging Fields (Not Replacing)

Use `"merge"` to update only the aggregated fields without losing others:

```javascript
db.sessions.aggregate([
  {
    $group: {
      _id: "$userId",
      sessionCount: { $sum: 1 },
      lastSeen: { $max: "$startedAt" }
    }
  },
  {
    $merge: {
      into: "userStats",
      on: "_id",
      whenMatched: "merge",
      whenNotMatched: "insert"
    }
  }
])
```

If `userStats` has other fields like `totalPurchases`, they are preserved.

## Using a Pipeline as whenMatched

Apply a custom pipeline when a matching document is found:

```javascript
db.events.aggregate([
  {
    $group: {
      _id: "$userId",
      newEvents: { $sum: 1 }
    }
  },
  {
    $merge: {
      into: "userActivity",
      on: "_id",
      whenMatched: [
        {
          $set: {
            totalEvents: { $add: ["$totalEvents", "$$new.newEvents"] },
            lastUpdated: "$$NOW"
          }
        }
      ],
      whenNotMatched: "insert"
    }
  }
])
```

`$$new` refers to the incoming document from the pipeline.

## Merging into a Different Database

```javascript
{
  $merge: {
    into: { db: "reporting", coll: "monthlySummary" },
    on: "_id",
    whenMatched: "replace",
    whenNotMatched: "insert"
  }
}
```

## Practical Use Case - Rolling Leaderboard

Update a leaderboard incrementally as new game results come in:

```javascript
db.gameResults.aggregate([
  { $match: { processed: false } },
  {
    $group: {
      _id: "$playerId",
      pointsEarned: { $sum: "$points" },
      gamesPlayed: { $sum: 1 }
    }
  },
  {
    $merge: {
      into: "leaderboard",
      on: "_id",
      whenMatched: [
        {
          $set: {
            totalPoints: { $add: ["$totalPoints", "$$new.pointsEarned"] },
            totalGames: { $add: ["$totalGames", "$$new.gamesPlayed"] }
          }
        }
      ],
      whenNotMatched: "insert"
    }
  }
])
```

## $merge vs $out

```text
Feature          | $out         | $merge
-----------------|--------------|------------------------------------------
Replaces target  | Always       | Only when whenMatched: "replace"
Merges results   | No           | Yes
Target must exist| No           | No (creates if needed)
Atomic           | Yes          | No (document-by-document)
```

Use `$merge` when you need incremental updates; use `$out` for full replacement.

## Summary

The `$merge` stage enables incremental, non-destructive writes of aggregation results to target collections. It supports fine-grained control over insert, merge, replace, and discard behaviors for both matched and unmatched documents. Use it to build incrementally updated summary collections, leaderboards, and reporting tables without costly full collection replacements.
