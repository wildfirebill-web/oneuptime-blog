# What Is $setWindowFields and When to Use It in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, setWindowFields, Window Function, Aggregation, Analytics

Description: The $setWindowFields stage in MongoDB applies window functions to compute running totals, rankings, and moving averages across ordered partitions of documents.

---

## Overview

`$setWindowFields` is a MongoDB aggregation stage introduced in version 5.0 that brings SQL-style window functions to MongoDB. A window function computes a value for each document based on a sliding "window" of neighboring documents, without collapsing the result set the way `$group` does. This makes it ideal for running totals, rankings, moving averages, and cumulative counts.

## Key Concepts

- **Partition** - Divide documents into groups (like `PARTITION BY` in SQL) using `partitionBy`
- **Sort order** - Define the ordering within each partition using `sortBy`
- **Window** - The range of documents that contribute to the calculation for each document (defined by `documents` or `range` bounds)
- **Output** - New fields added to each document with the computed values

## Syntax

```javascript
{
  $setWindowFields: {
    partitionBy: <expression>,   // optional grouping field
    sortBy: { <field>: 1 },      // required for most window functions
    output: {
      <newField>: {
        <windowFunction>: <args>,
        window: {
          documents: ["unbounded", "current"] // or range-based
        }
      }
    }
  }
}
```

## Example: Running Total of Sales

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { date: 1 },
      output: {
        runningTotal: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

Each document gets a `runningTotal` field that is the cumulative sum of `amount` within the same `region`, ordered by `date`.

## Example: Rank Salespeople

```javascript
db.salespeople.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$quarter",
      sortBy: { totalSales: -1 },
      output: {
        rank: { $rank: {} }
      }
    }
  }
])
```

## Example: 3-Period Moving Average

```javascript
db.metrics.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$serviceId",
      sortBy: { timestamp: 1 },
      output: {
        movingAvg: {
          $avg: "$responseTimeMs",
          window: { documents: [-2, 0] } // current + 2 preceding
        }
      }
    }
  }
])
```

## Available Window Functions

| Function | Description |
|---|---|
| `$sum` | Sum of values in the window |
| `$avg` | Average of values |
| `$min`, `$max` | Min/max value in the window |
| `$count` | Count of documents in the window |
| `$rank` | Rank with gaps for ties |
| `$denseRank` | Rank without gaps |
| `$documentNumber` | Sequential number within partition |
| `$first`, `$last` | First/last value in the window |
| `$shift` | Value from a document offset by N positions |
| `$expMovingAvg` | Exponential moving average |
| `$integral` | Integral over the window |
| `$derivative` | Rate of change |

## Window Bounds

**Document-based** - `["unbounded", "current"]` means all previous documents through the current one. `[-1, 1]` means one document before and after the current.

**Range-based** - Uses value ranges rather than document counts. Useful for time-series where documents may not be evenly spaced.

```javascript
window: { range: [-86400000, 0], unit: "millisecond" } // last 24 hours
```

## Performance Notes

`$setWindowFields` requires a sort within each partition. Ensure you have an index that supports the sort to avoid in-memory sorts. For large datasets, use `allowDiskUse: true`.

## Summary

`$setWindowFields` brings the power of SQL window functions to MongoDB, enabling running totals, rankings, and moving averages within partitioned, sorted data. It is the go-to stage for analytical queries where you need per-document calculations that consider neighboring documents.
