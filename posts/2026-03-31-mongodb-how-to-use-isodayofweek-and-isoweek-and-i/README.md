# How to Use $isoDayOfWeek, $isoWeek, and $isoWeekYear in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, ISO Date, Date Expressions, Database

Description: Learn how to use $isoDayOfWeek, $isoWeek, and $isoWeekYear in MongoDB aggregation to work with ISO 8601 week-based date calculations and grouping.

---

## Overview

MongoDB provides ISO 8601 compliant date operators - `$isoDayOfWeek`, `$isoWeek`, and `$isoWeekYear` - alongside the standard date operators. The ISO week system is widely used in business and finance contexts where weeks start on Monday and weeks are numbered according to the ISO 8601 standard, which can differ from the calendar year.

## Understanding ISO 8601 Week Numbering

In ISO 8601:
- Weeks start on Monday (day 1) and end on Sunday (day 7)
- Week 1 is the week containing the first Thursday of the year
- The ISO week year may differ from the calendar year for dates near the year boundary

This differs from `$dayOfWeek` (Sunday = 1) and `$week` (weeks start on Sunday).

## Using $isoDayOfWeek

The `$isoDayOfWeek` operator returns the day of the week as a number from 1 (Monday) to 7 (Sunday):

```javascript
db.orders.aggregate([
  {
    $project: {
      orderDate: 1,
      isoDayOfWeek: { $isoDayOfWeek: "$orderDate" },
      isWeekend: {
        $gte: [{ $isoDayOfWeek: "$orderDate" }, 6]
      }
    }
  }
])
```

Group orders by day of week to analyze weekday vs weekend patterns:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: { $isoDayOfWeek: "$saleDate" },
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])
```

A result where `_id: 1` = Monday, `_id: 7` = Sunday.

## Using $isoWeek

The `$isoWeek` operator returns the ISO week number (1-53) for a given date:

```javascript
db.metrics.aggregate([
  {
    $project: {
      date: 1,
      isoWeek: { $isoWeek: "$date" },
      isoWeekYear: { $isoWeekYear: "$date" }
    }
  }
])
```

Group data by ISO week for weekly reports:

```javascript
db.tickets.aggregate([
  {
    $group: {
      _id: {
        year: { $isoWeekYear: "$createdAt" },
        week: { $isoWeek: "$createdAt" }
      },
      openedTickets: { $sum: 1 },
      avgResolutionHours: { $avg: "$resolutionHours" }
    }
  },
  { $sort: { "_id.year": 1, "_id.week": 1 } }
])
```

## Using $isoWeekYear

The `$isoWeekYear` operator returns the ISO week-numbering year, which may differ from the calendar year for dates near the year boundary.

```javascript
db.deliveries.aggregate([
  {
    $project: {
      deliveryDate: 1,
      calendarYear: { $year: "$deliveryDate" },
      isoWeekYear: { $isoWeekYear: "$deliveryDate" },
      isoWeek: { $isoWeek: "$deliveryDate" }
    }
  },
  {
    $match: {
      $expr: { $ne: ["$calendarYear", "$isoWeekYear"] }
    }
  }
])
```

This finds dates where calendar year and ISO week year differ - typically the last few days of December or first few days of January.

## Timezone-Aware ISO Date Calculations

All ISO date operators support a timezone parameter:

```javascript
db.sessions.aggregate([
  {
    $group: {
      _id: {
        year: {
          $isoWeekYear: {
            date: "$startTime",
            timezone: "Europe/London"
          }
        },
        week: {
          $isoWeek: {
            date: "$startTime",
            timezone: "Europe/London"
          }
        }
      },
      sessions: { $sum: 1 }
    }
  }
])
```

## Building a Weekly Performance Dashboard

A complete example of a weekly KPI report using ISO week operators:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: {
        isoYear: { $isoWeekYear: "$orderDate" },
        isoWeek: { $isoWeek: "$orderDate" }
      },
      revenue: { $sum: "$total" },
      orders: { $sum: 1 },
      avgOrderValue: { $avg: "$total" }
    }
  },
  {
    $project: {
      weekLabel: {
        $concat: [
          { $toString: "$_id.isoYear" },
          "-W",
          {
            $cond: {
              if: { $lt: ["$_id.isoWeek", 10] },
              then: { $concat: ["0", { $toString: "$_id.isoWeek" }] },
              else: { $toString: "$_id.isoWeek" }
            }
          }
        ]
      },
      revenue: 1,
      orders: 1,
      avgOrderValue: { $round: ["$avgOrderValue", 2] }
    }
  },
  { $sort: { weekLabel: 1 } }
])
```

## Summary

MongoDB's ISO date operators `$isoDayOfWeek`, `$isoWeek`, and `$isoWeekYear` are essential for business reporting that follows the ISO 8601 calendar standard. Unlike standard week operators, they ensure consistent Monday-based week boundaries and correct handling of year-end weeks. Use `$isoWeekYear` instead of `$year` when grouping by ISO week to avoid misattributing late-December or early-January dates to the wrong year.
