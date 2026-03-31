# How to Use $sum and $avg in MongoDB Aggregation Group Accumulators

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $sum, $avg, Group Accumulators, Analytics

Description: Learn how to use $sum and $avg accumulators in MongoDB $group stages to calculate totals and averages across grouped documents.

---

## Overview

`$sum` and `$avg` are fundamental accumulator operators in MongoDB's aggregation framework. `$sum` computes the total of numeric values, and `$avg` computes the arithmetic mean. Both work in `$group` stages and in `$project`/`$addFields` to operate on arrays.

## Basic $sum Usage

In a `$group` stage, `$sum` adds up the specified field across all documents in the group:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: "$product",
      totalRevenue: { $sum: "$revenue" },
      totalQuantity: { $sum: "$quantity" }
    }
  }
])
```

Count documents in each group using `{ $sum: 1 }`:

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

## Basic $avg Usage

`$avg` computes the arithmetic mean, ignoring non-numeric and null values:

```javascript
db.reviews.aggregate([
  {
    $group: {
      _id: "$productId",
      averageRating: { $avg: "$rating" },
      reviewCount: { $sum: 1 }
    }
  },
  { $sort: { averageRating: -1 } }
])
```

## Grouping by Multiple Fields

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: { region: "$region", year: { $year: "$date" } },
      totalSales: { $sum: "$amount" },
      avgOrderValue: { $avg: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.region": 1 } }
])
```

## $sum on Arrays in $project

When used in a `$project` stage, `$sum` can add up elements of an array:

```javascript
db.students.insertMany([
  { name: "Alice", scores: [85, 92, 78, 95] },
  { name: "Bob", scores: [70, 65, 88, 72] }
])

db.students.aggregate([
  {
    $project: {
      name: 1,
      totalScore: { $sum: "$scores" },
      averageScore: { $avg: "$scores" }
    }
  }
])
// Result: [{ name: "Alice", totalScore: 350, averageScore: 87.5 }, ...]
```

## Conditional Sum with $cond

Count or sum only records that meet a condition:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalOrders: { $sum: 1 },
      completedOrders: {
        $sum: { $cond: [{ $eq: ["$status", "completed"] }, 1, 0] }
      },
      revenueFromCompleted: {
        $sum: {
          $cond: [{ $eq: ["$status", "completed"] }, "$amount", 0]
        }
      }
    }
  }
])
```

## Practical Example - Sales Dashboard Metrics

```javascript
db.transactions.aggregate([
  {
    $match: {
      date: {
        $gte: ISODate("2024-01-01"),
        $lt: ISODate("2025-01-01")
      }
    }
  },
  {
    $group: {
      _id: {
        month: { $month: "$date" },
        category: "$category"
      },
      totalRevenue: { $sum: "$amount" },
      avgTransactionValue: { $avg: "$amount" },
      transactionCount: { $sum: 1 },
      totalItemsSold: { $sum: "$itemCount" }
    }
  },
  {
    $project: {
      month: "$_id.month",
      category: "$_id.category",
      totalRevenue: { $round: ["$totalRevenue", 2] },
      avgTransactionValue: { $round: ["$avgTransactionValue", 2] },
      transactionCount: 1,
      totalItemsSold: 1
    }
  },
  { $sort: { month: 1, category: 1 } }
])
```

## Weighted Average with $sum and $divide

MongoDB does not have a built-in weighted average, but you can compute it with `$sum` and `$divide`:

```javascript
db.studentGrades.aggregate([
  {
    $group: {
      _id: "$studentId",
      weightedSum: {
        $sum: { $multiply: ["$grade", "$credits"] }
      },
      totalCredits: { $sum: "$credits" }
    }
  },
  {
    $project: {
      gpa: { $divide: ["$weightedSum", "$totalCredits"] }
    }
  }
])
```

## Summary

`$sum` totals numeric values across a group and can count documents with `{ $sum: 1 }`, while `$avg` computes the arithmetic mean ignoring nulls. Both can operate on arrays within `$project` stages. Using `$cond` with `$sum` enables conditional counting and totaling. Together, `$sum` and `$avg` are the backbone of most analytical aggregation queries in MongoDB.
