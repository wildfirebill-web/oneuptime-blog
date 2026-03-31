# How to Use $first and $last in MongoDB Aggregation Group Accumulators

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $first, $last, Group Accumulators

Description: Learn how to use $first and $last accumulators in MongoDB $group stages to retrieve the first and last values within each group.

---

## Overview

`$first` and `$last` are group accumulators in MongoDB's aggregation framework. They return the value of an expression for the first or last document in each group, respectively. The order of documents passed to `$group` depends on the pipeline stages that precede it - typically a `$sort` stage.

## Basic $first and $last Usage

```javascript
db.orders.aggregate([
  { $sort: { customerId: 1, orderDate: 1 } },
  {
    $group: {
      _id: "$customerId",
      firstOrder: { $first: "$orderDate" },
      lastOrder: { $last: "$orderDate" },
      firstAmount: { $first: "$amount" },
      lastAmount: { $last: "$amount" }
    }
  }
])
```

## Finding the Earliest and Latest Events

```javascript
db.events.insertMany([
  { userId: "u1", event: "login", timestamp: ISODate("2024-01-01T08:00:00Z") },
  { userId: "u1", event: "purchase", timestamp: ISODate("2024-01-01T09:30:00Z") },
  { userId: "u1", event: "logout", timestamp: ISODate("2024-01-01T10:00:00Z") },
  { userId: "u2", event: "login", timestamp: ISODate("2024-01-01T11:00:00Z") },
  { userId: "u2", event: "logout", timestamp: ISODate("2024-01-01T11:45:00Z") }
])

db.events.aggregate([
  { $sort: { userId: 1, timestamp: 1 } },
  {
    $group: {
      _id: "$userId",
      firstEvent: { $first: "$event" },
      lastEvent: { $last: "$event" },
      sessionStart: { $first: "$timestamp" },
      sessionEnd: { $last: "$timestamp" }
    }
  },
  {
    $addFields: {
      sessionDurationMinutes: {
        $divide: [
          { $subtract: ["$sessionEnd", "$sessionStart"] },
          60000
        ]
      }
    }
  }
])
```

## Getting the First/Last Complete Document

Use `$$ROOT` to capture the entire first or last document in a group:

```javascript
db.stockPrices.aggregate([
  { $sort: { symbol: 1, date: 1 } },
  {
    $group: {
      _id: "$symbol",
      openingRecord: { $first: "$$ROOT" },
      closingRecord: { $last: "$$ROOT" }
    }
  },
  {
    $project: {
      symbol: "$_id",
      openPrice: "$openingRecord.price",
      closePrice: "$closingRecord.price",
      priceChange: {
        $subtract: ["$closingRecord.price", "$openingRecord.price"]
      }
    }
  }
])
```

## Practical Example - Employee Hire and Latest Review

```javascript
db.employeeHistory.insertMany([
  { empId: 101, type: "hire", date: ISODate("2020-03-15"), note: "Initial hire" },
  { empId: 101, type: "review", date: ISODate("2021-03-15"), note: "Meets expectations" },
  { empId: 101, type: "review", date: ISODate("2022-03-15"), note: "Exceeds expectations" },
  { empId: 102, type: "hire", date: ISODate("2019-06-01"), note: "Initial hire" },
  { empId: 102, type: "review", date: ISODate("2022-06-01"), note: "Outstanding" }
])

db.employeeHistory.aggregate([
  { $sort: { empId: 1, date: 1 } },
  {
    $group: {
      _id: "$empId",
      hireDate: { $first: "$date" },
      latestReviewNote: { $last: "$note" },
      latestReviewDate: { $last: "$date" }
    }
  }
])
```

## $first and $last with Sorting on Multiple Fields

To get the first record per group by a specific sort field different from the group key:

```javascript
// Get the cheapest product per category
db.products.aggregate([
  { $sort: { category: 1, price: 1 } },
  {
    $group: {
      _id: "$category",
      cheapestProduct: { $first: "$name" },
      lowestPrice: { $first: "$price" },
      mostExpensiveProduct: { $last: "$name" },
      highestPrice: { $last: "$price" }
    }
  }
])
```

## $first in Window Functions (MongoDB 5.0+)

In MongoDB 5.0+, `$first` and `$last` also work as window function operators in `$setWindowFields`:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { date: 1 },
      output: {
        firstSaleOfRegion: {
          $first: "$amount",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

## Summary

`$first` and `$last` group accumulators return the value of an expression for the first or last document encountered in a group, making them useful for tracking earliest events, latest updates, opening/closing values, and chronological ordering within groups. Always pair them with a preceding `$sort` stage to ensure the order is deterministic and meaningful. Using `$$ROOT` with these accumulators allows capturing complete documents rather than individual fields.
