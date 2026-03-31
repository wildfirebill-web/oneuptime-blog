# How to Use $sum and $avg in MongoDB Aggregation Group Accumulators

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Group Accumulators, Analytics, Database

Description: Learn how to use $sum and $avg as group accumulators in MongoDB aggregation to calculate totals and averages across grouped documents efficiently.

---

## Overview

`$sum` and `$avg` are the most frequently used accumulators in MongoDB's `$group` stage. `$sum` computes the total of numeric values across a group, while `$avg` computes the arithmetic mean. Both operators work directly on field values, expressions, or even constants - making them versatile building blocks for analytical aggregations.

## Using $sum to Count and Total

The simplest use of `$sum` is counting documents in each group:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  }
])
```

Summing a numeric field:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: "$region",
      totalRevenue: { $sum: "$amount" },
      totalOrders: { $sum: 1 }
    }
  },
  { $sort: { totalRevenue: -1 } }
])
```

## Computing Multiple Sums in One Pass

Compute multiple aggregations simultaneously:

```javascript
db.invoices.aggregate([
  {
    $group: {
      _id: { year: { $year: "$date" }, month: { $month: "$date" } },
      totalGross: { $sum: "$grossAmount" },
      totalTax: { $sum: "$taxAmount" },
      totalNet: { $sum: "$netAmount" },
      invoiceCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

## Using $sum with Expressions

Sum a computed value rather than a raw field:

```javascript
db.lineItems.aggregate([
  {
    $group: {
      _id: "$orderId",
      orderTotal: {
        $sum: { $multiply: ["$price", "$quantity"] }
      }
    }
  }
])
```

## Using $avg to Calculate Averages

`$avg` computes the mean of all non-null numeric values in the group:

```javascript
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      averagePrice: { $avg: "$price" },
      averageRating: { $avg: "$rating" }
    }
  }
])
```

## Conditional $sum for Filtered Aggregation

Use `$cond` with `$sum` to conditionally count or sum subsets:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalOrders: { $sum: 1 },
      completedOrders: {
        $sum: { $cond: [{ $eq: ["$status", "completed"] }, 1, 0] }
      },
      completedRevenue: {
        $sum: {
          $cond: [
            { $eq: ["$status", "completed"] },
            "$amount",
            0
          ]
        }
      }
    }
  }
])
```

## Weighted Average with $sum

Compute a weighted average manually using two sums:

```javascript
db.reviews.aggregate([
  {
    $group: {
      _id: "$productId",
      totalWeightedScore: { $sum: { $multiply: ["$rating", "$weight"] } },
      totalWeight: { $sum: "$weight" }
    }
  },
  {
    $project: {
      weightedAverage: { $divide: ["$totalWeightedScore", "$totalWeight"] }
    }
  }
])
```

## Using $sum and $avg in $setWindowFields

For running totals and moving averages without collapsing the dataset:

```javascript
db.dailyRevenue.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { date: 1 },
      output: {
        cumulativeRevenue: {
          $sum: "$revenue",
          window: { documents: ["unbounded", "current"] }
        },
        movingAvg7Day: {
          $avg: "$revenue",
          window: { documents: [-6, "current"] }
        }
      }
    }
  }
])
```

## Summary

`$sum` and `$avg` are the foundation of numeric aggregation in MongoDB. Used in `$group` stages, they compute totals and averages across grouped documents. With expression arguments, they support computed values and conditional accumulation. In `$setWindowFields`, they enable running totals and rolling averages over sorted partitions - all without leaving the aggregation pipeline.
