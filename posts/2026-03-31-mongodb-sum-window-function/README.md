# How to Use $sum as a Window Function in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Window Function, Aggregation, Analytics, Sum

Description: Learn how to use $sum as a window function in MongoDB's $setWindowFields stage to compute running totals and cumulative sums over ordered partitions.

---

MongoDB's `$setWindowFields` stage, introduced in version 5.0, enables window function calculations - operations that compute values over a sliding window of documents without collapsing the result set. Using `$sum` as a window function lets you compute running totals, cumulative sums, and partition-level totals while retaining individual document rows.

## Basic Syntax

The `$setWindowFields` stage uses `$sum` within a `$window` specification:

```javascript
db.collection.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$fieldToGroupBy",
      sortBy: { fieldToSortBy: 1 },
      output: {
        resultField: {
          $sum: "$valueField",
          window: {
            documents: ["unbounded", "current"]
          }
        }
      }
    }
  }
]);
```

## Running Total (Cumulative Sum)

Compute a running total of order amounts per customer, ordered by date:

```javascript
db.orders.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$customerId",
      sortBy: { orderDate: 1 },
      output: {
        runningTotal: {
          $sum: "$amount",
          window: {
            documents: ["unbounded", "current"]
          }
        }
      }
    }
  },
  {
    $project: {
      customerId: 1,
      orderDate: 1,
      amount: 1,
      runningTotal: 1
    }
  }
]);
```

`documents: ["unbounded", "current"]` means "from the first document in the partition to the current document" - this produces a cumulative sum.

## Rolling Sum Over a Fixed Window

Calculate the 3-day rolling sum of sales for each store:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$storeId",
      sortBy: { saleDate: 1 },
      output: {
        rolling3DaySum: {
          $sum: "$revenue",
          window: {
            range: [-2, 0],
            unit: "day"
          }
        }
      }
    }
  }
]);
```

`range: [-2, 0]` with `unit: "day"` means "include documents from 2 days before the current document's date to the current document's date."

## Partition Total Without Collapsing Rows

Use `documents: ["unbounded", "unbounded"]` to compute the total for the entire partition, attached to every document in it:

```javascript
db.orders.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { orderDate: 1 },
      output: {
        regionTotal: {
          $sum: "$amount",
          window: {
            documents: ["unbounded", "unbounded"]
          }
        }
      }
    }
  },
  {
    $project: {
      region: 1,
      amount: 1,
      regionTotal: 1,
      percentOfRegionTotal: {
        $multiply: [
          { $divide: ["$amount", "$regionTotal"] },
          100
        ]
      }
    }
  }
]);
```

This allows you to compute each order's share of its region's total in a single pass.

## Fixed Document Window Sum

Sum over the previous 2 and next 2 documents (a 5-document window centered on current):

```javascript
db.metrics.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$metricName",
      sortBy: { timestamp: 1 },
      output: {
        smoothedSum: {
          $sum: "$value",
          window: {
            documents: [-2, 2]
          }
        }
      }
    }
  }
]);
```

## Summary

The `$sum` window function in MongoDB's `$setWindowFields` stage is a versatile tool for analytical queries. Use `documents: ["unbounded", "current"]` for cumulative running totals, `range` windows with `unit` for time-based rolling sums, and `documents: ["unbounded", "unbounded"]` for partition-wide totals that enable ratio calculations. All these patterns preserve individual document rows unlike `$group`, making them ideal for analytical dashboards and reporting pipelines.
