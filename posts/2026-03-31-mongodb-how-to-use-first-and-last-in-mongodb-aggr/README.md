# How to Use $first and $last in MongoDB Aggregation Group Accumulators

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Group Accumulators, Database

Description: Learn how to use $first and $last as group accumulators in MongoDB aggregation to retrieve the first and last values encountered in each group.

---

## Overview

The `$first` and `$last` accumulators in MongoDB's `$group` stage return the first and last values of a field within each group, respectively. Their result depends on the order of documents in the pipeline, so they are most useful when combined with a preceding `$sort` stage to produce meaningful first/last values.

## Using $first to Get the Earliest Value

`$first` returns the value of the expression for the first document in the group:

```javascript
db.orders.aggregate([
  { $sort: { orderDate: 1 } },
  {
    $group: {
      _id: "$customerId",
      firstOrderDate: { $first: "$orderDate" },
      firstOrderAmount: { $first: "$amount" },
      firstOrderId: { $first: "$_id" }
    }
  }
])
```

Because the documents are sorted by `orderDate` ascending before grouping, `$first` returns the earliest order per customer.

## Using $last to Get the Most Recent Value

`$last` returns the value from the last document in the group:

```javascript
db.sessions.aggregate([
  { $sort: { startTime: 1 } },
  {
    $group: {
      _id: "$userId",
      lastLoginTime: { $last: "$startTime" },
      lastIpAddress: { $last: "$ipAddress" },
      lastSessionId: { $last: "$_id" }
    }
  }
])
```

After sorting by `startTime` ascending, `$last` gives the most recent session per user.

## Combining $first and $last in the Same Group

Get both the earliest and most recent events in one pass:

```javascript
db.transactions.aggregate([
  { $sort: { date: 1 } },
  {
    $group: {
      _id: "$accountId",
      firstTransaction: { $first: "$date" },
      lastTransaction: { $last: "$date" },
      firstAmount: { $first: "$amount" },
      lastAmount: { $last: "$amount" },
      transactionCount: { $sum: 1 }
    }
  }
])
```

## Using $first and $last with Non-Scalar Fields

You can capture entire sub-documents as first/last:

```javascript
db.auditLog.aggregate([
  { $sort: { timestamp: 1 } },
  {
    $group: {
      _id: "$resourceId",
      firstAction: { $first: { action: "$action", actor: "$actor", timestamp: "$timestamp" } },
      lastAction: { $last: { action: "$action", actor: "$actor", timestamp: "$timestamp" } }
    }
  }
])
```

## Using $first Without Sort for Arbitrary Selection

When document order is not a concern - such as when all documents in the group have the same value for a field - `$first` is an efficient way to include that shared value in the output:

```javascript
db.products.aggregate([
  {
    $group: {
      _id: "$categoryId",
      categoryName: { $first: "$categoryName" },
      totalProducts: { $sum: 1 },
      avgPrice: { $avg: "$price" }
    }
  }
])
```

Here `categoryName` is the same for all products in a category, so `$first` is an efficient way to include it.

## $first and $last as Window Function Operators

In MongoDB 5.0+, `$first` and `$last` can also be used in `$setWindowFields` for window-based operations:

```javascript
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { date: 1 },
      output: {
        openPrice: {
          $first: "$price",
          window: { documents: ["unbounded", "current"] }
        },
        latestPrice: {
          $last: "$price",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

## Practical Example: Customer Lifecycle Summary

```javascript
db.orders.aggregate([
  { $sort: { orderDate: 1 } },
  {
    $group: {
      _id: "$customerId",
      acquisitionDate: { $first: "$orderDate" },
      mostRecentOrderDate: { $last: "$orderDate" },
      firstOrderValue: { $first: "$total" },
      lastOrderValue: { $last: "$total" },
      orderCount: { $sum: 1 },
      totalSpend: { $sum: "$total" }
    }
  },
  {
    $addFields: {
      lifetimeDays: {
        $dateDiff: {
          startDate: "$acquisitionDate",
          endDate: "$mostRecentOrderDate",
          unit: "day"
        }
      }
    }
  }
])
```

## Summary

The `$first` and `$last` group accumulators are simple but powerful tools for capturing boundary values within groups. Their behavior is order-dependent, so always precede a `$group` with a `$sort` stage when the first or last value is meaningful. For window-based first/last operations without collapsing groups, use them in `$setWindowFields` instead. Together they enable rich customer lifecycle, event boundary, and audit trail analyses.
