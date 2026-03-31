# How to Calculate the Sum of a Field Across Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $sum, Aggregation, Sum, $group

Description: Learn how to calculate the total sum of a numeric field across all documents or per group in MongoDB using the $sum accumulator in aggregation pipelines.

---

## Overview

MongoDB's `$sum` accumulator is the standard way to total a numeric field across documents. It works in `$group` stages for summing across documents, and in `$project` and `$addFields` stages for summing across fields within a single document.

## Total Sum Across All Documents

```javascript
// Total revenue across all completed orders
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: null,
      totalRevenue: { $sum: "$amount" }
    }
  }
]);

// Result: { "_id": null, "totalRevenue": 1250430.75 }
```

## Sum Per Group

```javascript
// Total revenue by customer
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalSpent: -1 } },
  { $limit: 10 }   // top 10 customers
]);
```

## Sum with Multiple Metrics

```javascript
// Revenue, discount totals, and order count per region
db.orders.aggregate([
  {
    $group: {
      _id: "$region",
      totalRevenue:  { $sum: "$amount" },
      totalDiscount: { $sum: "$discountAmount" },
      orderCount:    { $sum: 1 },
      avgOrder:      { $avg: "$amount" }
    }
  },
  { $sort: { totalRevenue: -1 } }
]);
```

## Sum with Conditional Multiplier

Use `$cond` to conditionally sum values:

```javascript
// Count approved vs rejected in one pass
db.applications.aggregate([
  {
    $group: {
      _id: null,
      approvedCount: {
        $sum: { $cond: [{ $eq: ["$status", "approved"] }, 1, 0] }
      },
      rejectedCount: {
        $sum: { $cond: [{ $eq: ["$status", "rejected"] }, 1, 0] }
      }
    }
  }
]);
```

## Sum of Nested Field

```javascript
// Total inventory value: sum of (price * quantity) per product
db.inventory.aggregate([
  {
    $project: {
      itemValue: { $multiply: ["$price", "$quantity"] }
    }
  },
  {
    $group: {
      _id: null,
      totalInventoryValue: { $sum: "$itemValue" }
    }
  }
]);

// Or in one stage using $sum in $group with an expression
db.inventory.aggregate([
  {
    $group: {
      _id: "$category",
      categoryValue: { $sum: { $multiply: ["$price", "$quantity"] } }
    }
  }
]);
```

## Sum of Array Fields Within a Document

```javascript
// Document: { "name": "Alice", "scores": [85, 90, 78] }
db.students.aggregate([
  {
    $project: {
      name: 1,
      totalScore: { $sum: "$scores" }
    }
  }
]);
// Result: { "name": "Alice", "totalScore": 253 }
```

## Sum Over a Time Range (Running Total)

Calculate cumulative sum using `$setWindowFields` (MongoDB 5.0+):

```javascript
db.dailyRevenue.aggregate([
  { $sort: { date: 1 } },
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        runningTotal: {
          $sum: "$revenue",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
]);
```

## Sum Per Month

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: {
        year:  { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      monthlyRevenue: { $sum: "$amount" },
      orderCount:     { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
]);
```

## Node.js Example

```javascript
async function getRevenueSummary(db, startDate, endDate) {
  const [result] = await db.collection("orders").aggregate([
    {
      $match: {
        status: "completed",
        createdAt: { $gte: startDate, $lte: endDate }
      }
    },
    {
      $group: {
        _id: null,
        totalRevenue:  { $sum: "$amount" },
        totalOrders:   { $sum: 1 },
        avgOrderValue: { $avg: "$amount" }
      }
    },
    {
      $project: {
        _id: 0,
        totalRevenue:  { $round: ["$totalRevenue", 2] },
        totalOrders: 1,
        avgOrderValue: { $round: ["$avgOrderValue", 2] }
      }
    }
  ]).toArray();

  return result || { totalRevenue: 0, totalOrders: 0, avgOrderValue: 0 };
}
```

## Summary

The `$sum` accumulator in MongoDB's aggregation pipeline totals numeric fields across grouped documents. Use `_id: null` to get a grand total, or group by a field for per-group totals. Combine `$sum` with `$multiply` or `$cond` for computed totals. In `$project`, `$sum` accepts an array of field paths to sum values within a single document. The `$setWindowFields` stage (MongoDB 5.0+) enables running totals over ordered data.
