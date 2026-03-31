# How to Use $first and $last as Window Functions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Analytics, Query

Description: Learn how to use $first and $last as window function operators in MongoDB aggregation pipelines to retrieve boundary values within ordered partitions.

---

## Window Functions in MongoDB

MongoDB 5.0 introduced `$setWindowFields` for window function support. The `$first` and `$last` operators, when used inside `$setWindowFields`, return the first or last value of a field within a defined window - without collapsing rows like group-stage accumulators do.

## Basic Syntax

```javascript
{
  $setWindowFields: {
    partitionBy: "$fieldToGroupBy",
    sortBy: { fieldToOrderBy: 1 },
    output: {
      firstValue: {
        $first: "$targetField",
        window: {
          documents: ["unbounded", "current"]  // from start to current row
        }
      },
      lastValue: {
        $last: "$targetField",
        window: {
          documents: ["current", "unbounded"]  // from current row to end
        }
      }
    }
  }
}
```

## Dataset Setup

```javascript
// Sample sales data
db.sales.insertMany([
  { salesperson: "Alice", region: "West", month: 1, revenue: 12000 },
  { salesperson: "Alice", region: "West", month: 2, revenue: 15000 },
  { salesperson: "Alice", region: "West", month: 3, revenue: 11000 },
  { salesperson: "Bob",   region: "East", month: 1, revenue: 9000  },
  { salesperson: "Bob",   region: "East", month: 2, revenue: 13000 },
  { salesperson: "Bob",   region: "East", month: 3, revenue: 16000 },
  { salesperson: "Carol", region: "West", month: 1, revenue: 8000  },
  { salesperson: "Carol", region: "West", month: 2, revenue: 10000 },
  { salesperson: "Carol", region: "West", month: 3, revenue: 14000 }
]);
```

## Example 1 - First Sale Revenue per Salesperson

Get each salesperson's revenue in their first recorded month alongside all their monthly records:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$salesperson",
      sortBy: { month: 1 },
      output: {
        firstMonthRevenue: {
          $first: "$revenue",
          window: { documents: ["unbounded", "unbounded"] }
        }
      }
    }
  },
  { $sort: { salesperson: 1, month: 1 } }
]);
```

```text
{ salesperson: "Alice", month: 1, revenue: 12000, firstMonthRevenue: 12000 }
{ salesperson: "Alice", month: 2, revenue: 15000, firstMonthRevenue: 12000 }
{ salesperson: "Alice", month: 3, revenue: 11000, firstMonthRevenue: 12000 }
{ salesperson: "Bob",   month: 1, revenue: 9000,  firstMonthRevenue: 9000  }
{ salesperson: "Bob",   month: 2, revenue: 13000, firstMonthRevenue: 9000  }
{ salesperson: "Bob",   month: 3, revenue: 16000, firstMonthRevenue: 9000  }
```

## Example 2 - Last Revenue in Each Region

Show the most recent monthly revenue for each region on every row:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { month: 1 },
      output: {
        latestRegionRevenue: {
          $last: "$revenue",
          window: { documents: ["unbounded", "unbounded"] }
        }
      }
    }
  },
  { $sort: { region: 1, month: 1 } }
]);
```

## Example 3 - Running First and Last for Trend Detection

Compare each row's value to the partition's opening and closing values:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$salesperson",
      sortBy: { month: 1 },
      output: {
        openingRevenue: {
          $first: "$revenue",
          window: { documents: ["unbounded", "unbounded"] }
        },
        closingRevenue: {
          $last: "$revenue",
          window: { documents: ["unbounded", "unbounded"] }
        }
      }
    }
  },
  {
    $addFields: {
      trend: {
        $cond: {
          if: { $gt: ["$closingRevenue", "$openingRevenue"] },
          then: "improving",
          else: "declining"
        }
      }
    }
  },
  { $sort: { salesperson: 1, month: 1 } }
]);
```

## Example 4 - Rolling First Value (Cumulative Window)

Use a documents window to get the first value from partition start to current row - useful for showing opening price in financial charts:

```javascript
// Stock price data
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { tradingDay: 1 },
      output: {
        dayOpen: {
          $first: "$price",
          window: { documents: ["unbounded", "current"] }
        },
        dayClose: {
          $last: "$price",
          window: { documents: ["current", "unbounded"] }
        }
      }
    }
  }
]);
```

## Example 5 - Range-Based Window with $first

Use a range window instead of documents window:

```javascript
// Get the first sale of the last 30 days relative to each row
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { saleDate: 1 },
      output: {
        firstInLast30Days: {
          $first: "$amount",
          window: {
            range: [-30, 0],
            unit: "day"
          }
        }
      }
    }
  }
]);
```

## Combining $first and $last with Other Window Functions

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$salesperson",
      sortBy: { month: 1 },
      output: {
        firstRevenue: {
          $first: "$revenue",
          window: { documents: ["unbounded", "unbounded"] }
        },
        lastRevenue: {
          $last: "$revenue",
          window: { documents: ["unbounded", "unbounded"] }
        },
        avgRevenue: {
          $avg: "$revenue",
          window: { documents: ["unbounded", "unbounded"] }
        },
        cumulativeRevenue: {
          $sum: "$revenue",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  },
  {
    $addFields: {
      changeFromStart: { $subtract: ["$lastRevenue", "$firstRevenue"] },
      changePercent: {
        $multiply: [
          { $divide: [
            { $subtract: ["$lastRevenue", "$firstRevenue"] },
            "$firstRevenue"
          ]},
          100
        ]
      }
    }
  }
]);
```

## Summary

`$first` and `$last` as window functions in MongoDB's `$setWindowFields` allow you to retrieve boundary values within partitions while preserving all original rows - unlike group-stage accumulators that collapse data. Use the `documents` window with `["unbounded", "unbounded"]` to get partition-wide first/last values, or scope the window to relative positions for rolling boundary calculations. They combine naturally with other window operators like `$sum`, `$avg`, and `$rank` for rich analytical queries.
