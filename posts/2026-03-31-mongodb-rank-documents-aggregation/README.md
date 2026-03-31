# How to Rank Documents in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Ranking, Analytics

Description: Learn how to rank documents in MongoDB using $rank, $denseRank, and $documentNumber operators in $setWindowFields for leaderboards and reporting.

---

Ranking documents is a common requirement in analytics - leaderboards, sales reports, and competition results all need ordered rank values. MongoDB 5.0 introduced native ranking operators inside `$setWindowFields`, removing the need for complex workarounds.

## Ranking Operators Available

MongoDB provides three ranking operators:

- `$rank` - assigns rank with gaps for ties (1, 1, 3, 4)
- `$denseRank` - assigns rank without gaps for ties (1, 1, 2, 3)
- `$documentNumber` - sequential row number regardless of ties (1, 2, 3, 4)

## Basic Rank Example

Given a `scores` collection:

```javascript
db.scores.insertMany([
  { player: "Alice", score: 950 },
  { player: "Bob",   score: 870 },
  { player: "Carol", score: 950 },
  { player: "Dave",  score: 720 }
])
```

Rank players by score descending:

```javascript
db.scores.aggregate([
  {
    $setWindowFields: {
      sortBy: { score: -1 },
      output: {
        rank:         { $rank: {} },
        denseRank:    { $denseRank: {} },
        rowNumber:    { $documentNumber: {} }
      }
    }
  },
  { $project: { _id: 0, player: 1, score: 1, rank: 1, denseRank: 1, rowNumber: 1 } }
])
```

Expected output:

```text
{ player: "Alice", score: 950, rank: 1, denseRank: 1, rowNumber: 1 }
{ player: "Carol", score: 950, rank: 1, denseRank: 1, rowNumber: 2 }
{ player: "Bob",   score: 870, rank: 3, denseRank: 2, rowNumber: 3 }
{ player: "Dave",  score: 720, rank: 4, denseRank: 3, rowNumber: 4 }
```

Alice and Carol tie - `$rank` skips to 3, `$denseRank` moves to 2.

## Ranking Within Groups (Partitioned Rank)

Use `partitionBy` to rank within separate groups:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { revenue: -1 },
      output: {
        regionalRank: { $rank: {} }
      }
    }
  },
  { $match: { regionalRank: { $lte: 3 } } }
])
```

This returns the top 3 sales reps per region, with each region ranked independently.

## Filtering by Rank - Top N Results

Combine `$setWindowFields` with a `$match` stage to retrieve only top-ranked documents:

```javascript
db.products.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { unitsSold: -1 },
      output: { rank: { $rank: {} } }
    }
  },
  { $match: { rank: { $lte: 5 } } },
  { $project: { name: 1, category: 1, unitsSold: 1, rank: 1 } },
  { $sort: { category: 1, rank: 1 } }
])
```

This returns the top 5 best-selling products per category.

## Percentile Rank Equivalent

To compute a percentile rank (normalized 0-1), combine `$documentNumber` with `$count`:

```javascript
db.scores.aggregate([
  {
    $setWindowFields: {
      sortBy: { score: 1 },
      output: {
        rowNum:     { $documentNumber: {} },
        totalCount: { $count: {}, window: { documents: ["unbounded", "unbounded"] } }
      }
    }
  },
  {
    $addFields: {
      percentileRank: {
        $divide: ["$rowNum", "$totalCount"]
      }
    }
  }
])
```

## Performance Tips

- Always include an index on the field used in `sortBy` to speed up the ranking sort.
- Use `$match` before `$setWindowFields` to reduce the working set.
- For very large collections, `$setWindowFields` may spill to disk - pass `{ allowDiskUse: true }` to `aggregate()`.

## Summary

MongoDB's `$rank`, `$denseRank`, and `$documentNumber` operators inside `$setWindowFields` provide SQL-style ranking natively. Use `partitionBy` to rank within subgroups, and follow with `$match` to filter to top-N results efficiently.
