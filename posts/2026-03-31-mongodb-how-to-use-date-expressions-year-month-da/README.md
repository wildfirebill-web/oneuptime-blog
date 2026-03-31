# How to Use Date Expressions in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Date Expressions, Database

Description: Learn how to use $year, $month, and $dayOfMonth date expressions in MongoDB aggregation pipelines to extract and group data by date components.

---

## Overview

MongoDB provides a rich set of date extraction operators in aggregation pipelines. The `$year`, `$month`, and `$dayOfMonth` operators let you extract individual components from date fields, enabling powerful time-based grouping, filtering, and reporting.

## Extracting Year, Month, and Day

The basic date extraction operators each accept a date field reference (or an object specifying the date and optional timezone):

```javascript
db.orders.aggregate([
  {
    $project: {
      orderDate: 1,
      year: { $year: "$orderDate" },
      month: { $month: "$orderDate" },
      day: { $dayOfMonth: "$orderDate" }
    }
  }
])
```

You can also extract additional components:

```javascript
db.events.aggregate([
  {
    $project: {
      eventDate: 1,
      hour: { $hour: "$eventDate" },
      minute: { $minute: "$eventDate" },
      second: { $second: "$eventDate" },
      millisecond: { $millisecond: "$eventDate" },
      dayOfWeek: { $dayOfWeek: "$eventDate" },
      dayOfYear: { $dayOfYear: "$eventDate" },
      week: { $week: "$eventDate" }
    }
  }
])
```

## Grouping by Date Components

The most common use case is grouping data by time periods:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$saleDate" },
        month: { $month: "$saleDate" }
      },
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

Group by day of week to find peak sales days:

```javascript
db.transactions.aggregate([
  {
    $group: {
      _id: { $dayOfWeek: "$createdAt" },
      avgAmount: { $avg: "$amount" },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])
```

Note: `$dayOfWeek` returns 1 (Sunday) through 7 (Saturday).

## Using Timezone-Aware Date Expressions

For applications serving multiple timezones, pass the timezone as part of the date expression:

```javascript
db.sessions.aggregate([
  {
    $project: {
      startTime: 1,
      localHour: {
        $hour: {
          date: "$startTime",
          timezone: "America/New_York"
        }
      },
      localDay: {
        $dayOfMonth: {
          date: "$startTime",
          timezone: "America/New_York"
        }
      }
    }
  }
])
```

## Filtering by Date Components with $match

Combine date extraction with `$match` to filter by specific time ranges:

```javascript
db.orders.aggregate([
  {
    $match: {
      $expr: {
        $and: [
          { $eq: [{ $year: "$orderDate" }, 2025] },
          { $in: [{ $month: "$orderDate" }, [11, 12]] }
        ]
      }
    }
  },
  {
    $group: {
      _id: { $month: "$orderDate" },
      total: { $sum: "$amount" }
    }
  }
])
```

## Building Date-Based Reports

A comprehensive monthly revenue report with year-over-year comparison:

```javascript
db.revenue.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$date" },
        month: { $month: "$date" }
      },
      total: { $sum: "$amount" },
      transactions: { $sum: 1 }
    }
  },
  {
    $project: {
      period: {
        $concat: [
          { $toString: "$_id.year" },
          "-",
          {
            $cond: {
              if: { $lt: ["$_id.month", 10] },
              then: { $concat: ["0", { $toString: "$_id.month" }] },
              else: { $toString: "$_id.month" }
            }
          }
        ]
      },
      total: 1,
      transactions: 1
    }
  },
  { $sort: { period: 1 } }
])
```

## Summary

MongoDB's date extraction operators like `$year`, `$month`, and `$dayOfMonth` are essential tools for time-based analytics. They integrate naturally with `$group`, `$project`, and `$match` stages to build comprehensive reporting pipelines. Always specify a timezone when your data spans multiple regions to ensure accurate date bucketing and comparisons.
