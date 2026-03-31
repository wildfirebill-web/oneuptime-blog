# How to Perform Aggregation Analytics in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Pipeline, Query

Description: Learn how to perform analytical queries in MongoDB using the aggregation pipeline to group, summarize, calculate trends, and generate reports directly from your collections.

---

MongoDB's aggregation pipeline processes documents through a series of stages to compute analytics results. Each stage transforms documents and passes them to the next, enabling complex analytical queries without moving data to a separate analytics system.

## Basic Grouping and Summaries

Compute total revenue and order count by category:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: "$category",
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$amount" }
    }
  },
  { $sort: { totalRevenue: -1 } },
  { $limit: 10 }
])
```

## Time Series Analytics

Aggregate orders by month to analyze trends:

```javascript
db.orders.aggregate([
  { $match: { status: "completed", createdAt: { $gte: new Date("2025-01-01") } } },
  {
    $group: {
      _id: {
        year: { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      revenue: { $sum: "$amount" },
      orders: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

## Calculate Percentiles with $percentile

MongoDB 7.0 introduced `$percentile` for analytical statistics:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$category",
      p50: { $percentile: { input: "$amount", p: [0.5], method: "approximate" } },
      p95: { $percentile: { input: "$amount", p: [0.95], method: "approximate" } },
      p99: { $percentile: { input: "$amount", p: [0.99], method: "approximate" } }
    }
  }
])
```

## Window Functions with $setWindowFields

Compute running totals and moving averages:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { date: 1 },
      output: {
        runningTotal: {
          $sum: "$revenue",
          window: { documents: ["unbounded", "current"] }
        },
        movingAvg7d: {
          $avg: "$revenue",
          window: { range: [-6, 0], unit: "day" }
        }
      }
    }
  }
])
```

## Funnel Analysis

Track user progression through steps:

```javascript
db.events.aggregate([
  { $match: { eventDate: { $gte: new Date("2025-01-01") } } },
  {
    $group: {
      _id: "$userId",
      events: { $addToSet: "$eventType" }
    }
  },
  {
    $group: {
      _id: null,
      viewed: { $sum: { $cond: [{ $in: ["view", "$events"] }, 1, 0] } },
      addedToCart: { $sum: { $cond: [{ $in: ["add_to_cart", "$events"] }, 1, 0] } },
      purchased: { $sum: { $cond: [{ $in: ["purchase", "$events"] }, 1, 0] } }
    }
  }
])
```

## Optimize Aggregation Performance

Place `$match` and `$limit` as early as possible in the pipeline to reduce the number of documents processed by later stages. Ensure the `$match` stage can use an index:

```javascript
db.orders.createIndex({ status: 1, createdAt: -1, category: 1 })
```

Use `explain()` to verify index usage:

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$category", total: { $sum: "$amount" } } }
])
```

## Summary

MongoDB's aggregation pipeline covers a wide range of analytical workloads directly in the database. Use `$group` for summaries, `$setWindowFields` for running totals and moving averages, and `$percentile` for statistical distributions. Always place `$match` stages early and verify index usage with `explain()` to keep analytical queries fast.
