# What Is $setWindowFields and When to Use It in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, SetWindowFields, Window Function, Analytics

Description: Learn what MongoDB's $setWindowFields aggregation stage does and when to use it to compute running totals, moving averages, and rankings over ordered data sets.

---

## What Is $setWindowFields

`$setWindowFields` is an aggregation stage introduced in MongoDB 5.0 that enables window functions - computations performed over a defined range (window) of documents relative to the current document. This is equivalent to SQL's `OVER (PARTITION BY ... ORDER BY ... ROWS BETWEEN ...)` syntax.

Window functions let you compute:
- Running totals and cumulative sums
- Moving averages
- Rank and dense rank
- Lead and lag values
- Document number within a partition

## Basic Syntax

```javascript
{
  $setWindowFields: {
    partitionBy: <expression>,    // optional - group documents
    sortBy: { <field>: 1 | -1 }, // required for most window ops
    output: {
      <newField>: {
        <windowOperator>: <expression>,
        window: {
          documents: [<lower>, <upper>]  // or
          range: [<lower>, <upper>]
        }
      }
    }
  }
}
```

Window bounds can be:
- `"unbounded"` - from the start/end of the partition
- `"current"` - the current document
- An integer - offset from current document

## Example 1 - Running Total

Calculate a cumulative sum of sales by date:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { date: 1 },
      output: {
        cumulativeSales: {
          $sum: "$amount",
          window: {
            documents: ["unbounded", "current"]
          }
        }
      }
    }
  }
]);
```

Each document gets a `cumulativeSales` field showing the total sales up to and including that date within the same region.

## Example 2 - 7-Day Moving Average

```javascript
db.metrics.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$service",
      sortBy: { date: 1 },
      output: {
        movingAvgLatency: {
          $avg: "$p99LatencyMs",
          window: {
            documents: [-6, 0]  // current + 6 preceding = 7 days
          }
        }
      }
    }
  }
]);
```

## Example 3 - Rank Within a Partition

Rank products by revenue within each category:

```javascript
db.products.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { revenue: -1 },
      output: {
        rank: { $rank: {} },
        denseRank: { $denseRank: {} },
        rowNumber: { $documentNumber: {} }
      }
    }
  },
  { $match: { rank: { $lte: 3 } } }  // top 3 per category
]);
```

Difference between rank operators:
- `$rank` - skips numbers on ties (1, 1, 3)
- `$denseRank` - no gaps on ties (1, 1, 2)
- `$documentNumber` - sequential, no regard for ties (1, 2, 3)

## Example 4 - Lag and Lead Values

Compare current month's sales to the previous month:

```javascript
db.monthlySales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$productId",
      sortBy: { yearMonth: 1 },
      output: {
        prevMonthSales: {
          $shift: {
            output: "$sales",
            by: -1,          // 1 document before current
            default: null
          }
        },
        nextMonthSales: {
          $shift: {
            output: "$sales",
            by: 1,
            default: null
          }
        }
      }
    }
  },
  {
    $addFields: {
      momGrowth: {
        $cond: [
          { $and: [{ $ne: ["$prevMonthSales", null] }, { $ne: ["$prevMonthSales", 0] }] },
          { $divide: [{ $subtract: ["$sales", "$prevMonthSales"] }, "$prevMonthSales"] },
          null
        ]
      }
    }
  }
]);
```

## Example 5 - Range-Based Windows

Use a range window to compute an average over a numeric range rather than document count:

```javascript
db.prices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { timestamp: 1 },
      output: {
        avgPriceLastHour: {
          $avg: "$price",
          window: {
            range: [-3600000, 0],  // last 3600000ms = 1 hour
            unit: "millisecond"
          }
        }
      }
    }
  }
]);
```

## Supported Window Operators

| Operator | Description |
|---|---|
| `$sum`, `$avg`, `$min`, `$max` | Standard accumulators over window |
| `$count` | Count documents in window |
| `$rank`, `$denseRank` | Ranking within partition |
| `$documentNumber` | Sequential row number |
| `$shift` | Lead/lag value |
| `$first`, `$last` | First or last value in window |
| `$derivative` | Rate of change |
| `$integral` | Area under a curve |
| `$covariancePop`, `$covarianceSamp` | Statistical covariance |
| `$expMovingAvg` | Exponential moving average |

## When to Use $setWindowFields

Use `$setWindowFields` when you need:
- Analytics over ordered time series data
- Rankings or percentiles within groups
- Comparisons to prior/next periods
- Cumulative metrics without self-joins

Before MongoDB 5.0, these patterns required complex `$group` + `$lookup` self-joins. `$setWindowFields` replaces that complexity with declarative, efficient window computations.

## Summary

`$setWindowFields` brings SQL-style window function capabilities to the MongoDB aggregation pipeline. It supports running totals, moving averages, rankings, and lead/lag comparisons through document-based or range-based windows, partitioned by any expression and ordered by any field. It is the preferred approach for time series analytics and leaderboard computations starting with MongoDB 5.0.
