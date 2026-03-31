# How to Rank Documents with $rank and $denseRank in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Ranking, Analytics

Description: Learn how to rank MongoDB documents using $rank and $denseRank window operators inside $setWindowFields to compute position-based rankings across sorted partitions.

---

MongoDB's `$rank` and `$denseRank` operators, available inside `$setWindowFields`, let you assign ordinal positions to documents based on their sorted order within a partition. They behave identically to SQL's `RANK()` and `DENSE_RANK()` window functions.

## Difference Between $rank and $denseRank

Both operators assign a rank number based on sort order, but they handle ties differently:

- `$rank` - when documents tie, they share a rank, and the next rank skips numbers (1, 1, 3, 4)
- `$denseRank` - when documents tie, they share a rank, and the next rank is consecutive (1, 1, 2, 3)

## Basic $rank Example

Rank sales representatives by revenue:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      sortBy: { revenue: -1 },
      output: {
        rank: {
          $rank: {}
        }
      }
    }
  },
  { $sort: { rank: 1 } }
])
```

Sample output:

```json
[
  { "name": "Alice", "revenue": 95000, "rank": 1 },
  { "name": "Bob", "revenue": 87000, "rank": 2 },
  { "name": "Carol", "revenue": 87000, "rank": 2 },
  { "name": "David", "revenue": 72000, "rank": 4 }
]
```

Alice and Bob share rank 2; the next rank is 4 (skips 3).

## Using $denseRank for Consecutive Ranking

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      sortBy: { revenue: -1 },
      output: {
        denseRank: {
          $denseRank: {}
        }
      }
    }
  },
  { $sort: { denseRank: 1 } }
])
```

Output with `$denseRank`:

```json
[
  { "name": "Alice", "revenue": 95000, "denseRank": 1 },
  { "name": "Bob", "revenue": 87000, "denseRank": 2 },
  { "name": "Carol", "revenue": 87000, "denseRank": 2 },
  { "name": "David", "revenue": 72000, "denseRank": 3 }
]
```

The same tie at rank 2, but David gets rank 3 instead of 4.

## Ranking Within Partitions

Use the `partitionBy` option to rank within groups:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { revenue: -1 },
      output: {
        regionRank: {
          $rank: {}
        },
        regionDenseRank: {
          $denseRank: {}
        }
      }
    }
  },
  { $sort: { region: 1, regionRank: 1 } }
])
```

This ranks each sales rep within their region independently, so the top performer in each region gets rank 1.

## Filtering Top-N Per Partition

Combine `$rank` with `$match` to retrieve the top performers per group:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$department",
      sortBy: { score: -1 },
      output: {
        departmentRank: { $rank: {} }
      }
    }
  },
  {
    $match: { departmentRank: { $lte: 3 } }
  },
  { $sort: { department: 1, departmentRank: 1 } }
])
```

This returns the top 3 employees per department by score - a common analytics requirement.

## Combining $rank with Other Window Operators

Add multiple output fields in a single `$setWindowFields` stage:

```javascript
db.products.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { sales: -1 },
      output: {
        salesRank: { $rank: {} },
        runningTotal: {
          $sum: "$sales",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

## Summary

`$rank` and `$denseRank` inside `$setWindowFields` bring SQL-style ranking to MongoDB aggregation pipelines. Use `$rank` when gaps in the rank sequence are acceptable and desired, and `$denseRank` when consecutive rank numbers are required despite ties. The `partitionBy` option enables independent rankings within groups, and combining these operators with `$match` is an efficient way to query top-N records per partition.
