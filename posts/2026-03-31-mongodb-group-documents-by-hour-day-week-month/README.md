# How to Group Documents by Hour, Day, Week, or Month in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Group, Date, Time Series

Description: Learn how to group MongoDB documents by time intervals - hour, day, week, or month - using $group with date operators and $dateTrunc.

---

## Overview

Time-based grouping is fundamental to analytics dashboards, trend analysis, and reporting. MongoDB provides several approaches depending on your version: `$dateTrunc` (5.0+) for clean interval bucketing, or date part extraction operators for older versions.

## Grouping by Day Using $dateTrunc (MongoDB 5.0+)

`$dateTrunc` truncates a date to a boundary, making it ideal as a group key.

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: {
        $dateTrunc: { date: "$createdAt", unit: "day" }
      },
      dailyRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { "_id": 1 } }
]);
```

## Grouping by Hour

```javascript
db.events.aggregate([
  {
    $group: {
      _id: {
        $dateTrunc: { date: "$timestamp", unit: "hour" }
      },
      eventCount: { $sum: 1 }
    }
  },
  { $sort: { "_id": 1 } }
]);
```

## Grouping by Week

```javascript
db.pageviews.aggregate([
  {
    $group: {
      _id: {
        $dateTrunc: { date: "$viewedAt", unit: "week" }
      },
      weeklyViews: { $sum: 1 }
    }
  }
]);
```

## Grouping by Month (Pre-5.0 Approach)

Extract date parts using `$year` and `$month`.

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: {
        year:  { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      monthlyRevenue: { $sum: "$amount" },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
]);
```

## Grouping by Day of Week

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { $dayOfWeek: "$createdAt" },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id": 1 } }
]);
```

Day 1 is Sunday, 7 is Saturday.

## Adding Timezone Support

When end-users are in a specific timezone, group by calendar days in that timezone.

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: {
        $dateTrunc: {
          date: "$createdAt",
          unit: "day",
          timezone: "America/Los_Angeles"
        }
      },
      revenue: { $sum: "$amount" }
    }
  }
]);
```

## Projecting a Formatted Label

Add a readable label to each bucket.

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { $dateTrunc: { date: "$createdAt", unit: "month" } },
      revenue: { $sum: "$amount" }
    }
  },
  {
    $project: {
      month: {
        $dateToString: { format: "%Y-%m", date: "$_id" }
      },
      revenue: 1
    }
  },
  { $sort: { month: 1 } }
]);
```

## Summary

Use `$dateTrunc` (MongoDB 5.0+) to group by time intervals cleanly with optional timezone support. For older versions, extract date parts with `$year`, `$month`, `$dayOfMonth`, or `$hour` as group keys. Add `$sort` and `$dateToString` to produce readable, ordered results.
