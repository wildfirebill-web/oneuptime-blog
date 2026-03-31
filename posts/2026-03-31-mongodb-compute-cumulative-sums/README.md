# How to Compute Cumulative Sums in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Cumulative, Window Function, Analytics

Description: Compute running totals and cumulative sums in MongoDB using $setWindowFields with unbounded window frames for time-series and financial analysis.

---

## Overview

Cumulative sums - also called running totals - are common in financial dashboards, sales tracking, and time-series analysis. MongoDB 5.0 introduced `$setWindowFields`, which supports window functions similar to SQL's `SUM() OVER (ORDER BY ...)`, allowing you to compute running totals without leaving the aggregation pipeline.

## Sample Data

```javascript
const transaction = {
  _id: ObjectId(),
  date: new Date("2026-03-15"),
  amount: 250.0,
  category: "sales",
};
```

## Cumulative Sum with $setWindowFields

Compute a running total ordered by date within the entire collection.

```javascript
const runningTotals = await db.collection("transactions").aggregate([
  { $match: { category: "sales" } },
  { $sort: { date: 1 } },
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { date: 1 },
      output: {
        cumulativeAmount: {
          $sum: "$amount",
          window: {
            documents: ["unbounded", "current"],
          },
        },
      },
    },
  },
  {
    $project: {
      date: 1,
      amount: 1,
      cumulativeAmount: 1,
    },
  },
]).toArray();
```

## Cumulative Sum Per Partition

Compute independent running totals for each category within the same pipeline.

```javascript
const byCategory = await db.collection("transactions").aggregate([
  { $sort: { category: 1, date: 1 } },
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { date: 1 },
      output: {
        runningTotal: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] },
        },
        movingAvg7: {
          $avg: "$amount",
          window: { documents: [-6, "current"] },
        },
      },
    },
  },
]).toArray();
```

## Daily Cumulative Sum with $group Then Window

Group by date first to get daily totals, then apply the window function.

```javascript
const dailyCumulative = await db.collection("transactions").aggregate([
  { $match: { date: { $gte: new Date("2026-01-01") } } },
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m-%d", date: "$date" } },
      dailyTotal: { $sum: "$amount" },
    },
  },
  { $sort: { _id: 1 } },
  {
    $setWindowFields: {
      sortBy: { _id: 1 },
      output: {
        cumulativeTotal: {
          $sum: "$dailyTotal",
          window: { documents: ["unbounded", "current"] },
        },
      },
    },
  },
  {
    $project: {
      date: "$_id",
      dailyTotal: 1,
      cumulativeTotal: 1,
      _id: 0,
    },
  },
]).toArray();
```

## Cumulative Count

Count documents cumulatively to track signups or event growth over time.

```javascript
const cumulativeSignups = await db.collection("users").aggregate([
  { $sort: { createdAt: 1 } },
  {
    $setWindowFields: {
      sortBy: { createdAt: 1 },
      output: {
        totalUsers: {
          $sum: 1,
          window: { documents: ["unbounded", "current"] },
        },
      },
    },
  },
  {
    $project: {
      createdAt: 1,
      totalUsers: 1,
    },
  },
]).toArray();
```

## Summary

MongoDB's `$setWindowFields` operator with `documents: ["unbounded", "current"]` computes cumulative sums natively inside the aggregation pipeline. Partition by a dimension like category or user ID to compute independent running totals, and combine with `$group` to pre-aggregate daily totals before applying the window for efficient large-dataset processing.
