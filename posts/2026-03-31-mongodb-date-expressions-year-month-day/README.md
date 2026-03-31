# How to Use Date Expressions ($year, $month, $dayOfMonth) in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Date, $year, $month

Description: Learn how to extract date components like year, month, day, hour, and minute from date fields in MongoDB aggregation using $year, $month, $dayOfMonth, and more.

---

MongoDB provides a family of date extraction operators that let you decompose a Date value into individual components inside an aggregation pipeline. These are essential for time-based grouping, reporting, and filtering.

## The Date Extraction Operators

| Operator | Returns |
|----------|---------|
| `$year` | 4-digit year (e.g., 2026) |
| `$month` | Month 1-12 |
| `$dayOfMonth` | Day of month 1-31 |
| `$dayOfWeek` | Day of week 1 (Sunday) to 7 (Saturday) |
| `$dayOfYear` | Day of year 1-366 |
| `$hour` | Hour 0-23 |
| `$minute` | Minute 0-59 |
| `$second` | Second 0-60 |
| `$millisecond` | Millisecond 0-999 |
| `$week` | Week number 0-53 |

## Basic Extraction

```js
db.orders.aggregate([
  {
    $project: {
      orderYear:  { $year:  "$createdAt" },
      orderMonth: { $month: "$createdAt" },
      orderDay:   { $dayOfMonth: "$createdAt" },
      orderHour:  { $hour: "$createdAt" }
    }
  }
]);
```

## Grouping by Month and Year

A common use case is aggregating sales by month:

```js
db.orders.aggregate([
  {
    $group: {
      _id: {
        year:  { $year:  "$createdAt" },
        month: { $month: "$createdAt" }
      },
      totalRevenue: { $sum: "$amount" },
      orderCount:   { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
]);
```

## Grouping by Day of Week

Analyze which days of the week have the most activity:

```js
db.events.aggregate([
  {
    $group: {
      _id: { $dayOfWeek: "$timestamp" },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id": 1 } }
]);
```

Days map as: 1=Sunday, 2=Monday, ..., 7=Saturday.

## Using Timezone

All date operators accept a timezone option to convert to a local time before extraction:

```js
db.orders.aggregate([
  {
    $project: {
      localHour: {
        $hour: {
          date: "$createdAt",
          timezone: "America/New_York"
        }
      },
      localDay: {
        $dayOfMonth: {
          date: "$createdAt",
          timezone: "America/New_York"
        }
      }
    }
  }
]);
```

## Building a Hourly Histogram

Find peak hours for a service:

```js
db.requests.aggregate([
  {
    $group: {
      _id: {
        $hour: { date: "$timestamp", timezone: "UTC" }
      },
      requestCount: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } },
  {
    $project: {
      hour: "$_id",
      requestCount: 1,
      _id: 0
    }
  }
]);
```

## Filtering by Date Component

Use date operators inside `$match` with `$expr` to filter documents:

```js
db.orders.aggregate([
  {
    $match: {
      $expr: {
        $and: [
          { $eq: [{ $year: "$createdAt" }, 2026] },
          { $eq: [{ $month: "$createdAt" }, 3] }
        ]
      }
    }
  }
]);
```

## Combining with $dateToString

Format the extracted components into a readable string:

```js
db.orders.aggregate([
  {
    $project: {
      monthLabel: {
        $dateToString: { format: "%B %Y", date: "$createdAt" }
      }
    }
  }
]);
```

## Summary

MongoDB's date extraction operators (`$year`, `$month`, `$dayOfMonth`, `$hour`, etc.) make time-based analytics straightforward in aggregation pipelines. Use the `timezone` option to handle local time conversions correctly, and combine these operators with `$group` to build time-bucketed reports without needing any application-side date processing.
