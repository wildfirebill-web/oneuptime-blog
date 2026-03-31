# How to Calculate Running Totals in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Analytics, Pipeline

Description: Learn how to calculate running totals and cumulative sums in MongoDB using $setWindowFields with $sum and the unbounded window frame.

---

A running total (cumulative sum) computes the sum of a field from the first record up to the current record in an ordered sequence. MongoDB 5.0+ introduced `$setWindowFields` for window function calculations, making running totals straightforward and efficient.

## Using $setWindowFields for Running Totals

`$setWindowFields` processes documents in a defined window relative to each document:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: null,          // no partition - single window for all docs
      sortBy: { saleDate: 1 },   // sort order for the running total
      output: {
        runningTotal: {
          $sum: "$amount",
          window: {
            documents: ["unbounded", "current"]
            // from the first document to the current document
          }
        }
      }
    }
  },
  {
    $project: {
      saleDate: 1,
      amount: 1,
      runningTotal: 1
    }
  }
])
```

Sample output:

```json
[
  { "saleDate": "2024-01-01", "amount": 100, "runningTotal": 100 },
  { "saleDate": "2024-01-02", "amount": 250, "runningTotal": 350 },
  { "saleDate": "2024-01-03", "amount": 75,  "runningTotal": 425 }
]
```

## Partitioned Running Totals

Compute running totals separately for each partition (e.g., per product category):

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { saleDate: 1 },
      output: {
        categoryRunningTotal: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

Each category gets its own running total, starting fresh from zero.

## Running Count and Running Average

Multiple running calculations can be added to a single `$setWindowFields` stage:

```javascript
db.pageviews.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$pageId",
      sortBy: { date: 1 },
      output: {
        cumulativeViews: {
          $sum: "$views",
          window: { documents: ["unbounded", "current"] }
        },
        cumulativeAvgViews: {
          $avg: "$views",
          window: { documents: ["unbounded", "current"] }
        },
        rowNumber: {
          $documentNumber: {}
        }
      }
    }
  }
])
```

## Moving Sum (Rolling Window)

Instead of an unbounded window, use a fixed range for a rolling total:

```javascript
// 7-day rolling total
db.daily_revenue.aggregate([
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        rollingSevenDayTotal: {
          $sum: "$revenue",
          window: { documents: [-6, "current"] }
            // current doc + 6 previous = 7 docs
        }
      }
    }
  }
])
```

## Range-Based Window (by Value)

Use a range window when the sort field is numeric or date and you want a value-based range:

```javascript
db.measurements.aggregate([
  {
    $setWindowFields: {
      sortBy: { timestamp: 1 },
      output: {
        oneHourRollingSum: {
          $sum: "$value",
          window: {
            range: [-3600000, "current"],  // 1 hour in milliseconds
            unit: "millisecond"
          }
        }
      }
    }
  }
])
```

## Pre-5.0 Alternative with $group and $unwind

Before `$setWindowFields` (MongoDB < 5.0), running totals required a more complex pattern:

```javascript
db.sales.aggregate([
  { $sort: { saleDate: 1 } },
  {
    $group: {
      _id: null,
      docs: { $push: "$$ROOT" }
    }
  },
  {
    $project: {
      docs: {
        $reduce: {
          input: "$docs",
          initialValue: { total: 0, items: [] },
          in: {
            total: { $add: ["$$value.total", "$$this.amount"] },
            items: {
              $concatArrays: [
                "$$value.items",
                [{ $mergeObjects: ["$$this", { runningTotal: { $add: ["$$value.total", "$$this.amount"] } }] }]
              ]
            }
          }
        }
      }
    }
  },
  { $unwind: "$docs.items" },
  { $replaceRoot: { newRoot: "$docs.items" } }
])
```

## Summary

Use `$setWindowFields` with `$sum` and `window: { documents: ["unbounded", "current"] }` to calculate running totals in MongoDB 5.0+. Use `partitionBy` to compute separate running totals per group. For rolling windows, specify a fixed document range or a time-based range. The `$setWindowFields` stage preserves all input documents and adds new computed fields, unlike `$group` which collapses documents.
