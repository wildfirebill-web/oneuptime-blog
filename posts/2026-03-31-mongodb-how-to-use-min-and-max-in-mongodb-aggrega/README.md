# How to Use $min and $max in MongoDB Aggregation Group Accumulators

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Group Accumulators, Analytics, Database

Description: Learn how to use $min and $max as group accumulators in MongoDB aggregation to find the minimum and maximum values of fields across grouped documents.

---

## Overview

The `$min` and `$max` accumulators in MongoDB's `$group` stage return the minimum and maximum values of a field or expression across all documents in a group. They work with numbers, dates, strings, and any BSON type that supports comparison - making them versatile for a wide range of analytical use cases.

## Using $min to Find the Minimum Value

`$min` returns the smallest value in the group:

```javascript
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      lowestPrice: { $min: "$price" },
      cheapestProduct: { $min: "$name" }
    }
  }
])
```

Note: For strings, `$min` uses lexicographic ordering. For dates, it returns the earliest date.

## Using $max to Find the Maximum Value

`$max` returns the largest value in the group:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      largestOrder: { $max: "$amount" },
      mostRecentOrder: { $max: "$orderDate" }
    }
  }
])
```

## Combining $min and $max for Range Analysis

Find the value range (spread) per group:

```javascript
db.sensors.aggregate([
  {
    $group: {
      _id: { sensor: "$sensorId", date: { $dateToString: { format: "%Y-%m-%d", date: "$timestamp" } } },
      minTemp: { $min: "$temperature" },
      maxTemp: { $max: "$temperature" },
      avgTemp: { $avg: "$temperature" },
      readings: { $sum: 1 }
    }
  },
  {
    $addFields: {
      tempRange: { $subtract: ["$maxTemp", "$minTemp"] }
    }
  },
  { $sort: { tempRange: -1 } }
])
```

## Using $min and $max on Dates

Find the earliest and latest dates per group:

```javascript
db.employeeAudit.aggregate([
  {
    $group: {
      _id: "$employeeId",
      firstActivity: { $min: "$timestamp" },
      lastActivity: { $max: "$timestamp" }
    }
  },
  {
    $addFields: {
      activeDays: {
        $dateDiff: {
          startDate: "$firstActivity",
          endDate: "$lastActivity",
          unit: "day"
        }
      }
    }
  }
])
```

## Using $min and $max with Expressions

Apply expressions before finding the min/max:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$productId",
      maxLineTotal: {
        $max: { $multiply: ["$price", "$quantity"] }
      },
      minLineTotal: {
        $min: { $multiply: ["$price", "$quantity"] }
      }
    }
  }
])
```

## Using $min and $max as Window Functions

In MongoDB 5.0+, use `$min` and `$max` in `$setWindowFields` for running min/max without collapsing documents:

```javascript
db.stockPrices.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$ticker",
      sortBy: { date: 1 },
      output: {
        allTimeHigh: {
          $max: "$price",
          window: { documents: ["unbounded", "current"] }
        },
        allTimeLow: {
          $min: "$price",
          window: { documents: ["unbounded", "current"] }
        },
        rolling30DayHigh: {
          $max: "$price",
          window: { documents: [-29, "current"] }
        }
      }
    }
  }
])
```

## Practical Example: Category Price Intelligence

```javascript
db.products.aggregate([
  {
    $group: {
      _id: "$categoryId",
      minPrice: { $min: "$price" },
      maxPrice: { $max: "$price" },
      avgPrice: { $avg: "$price" },
      productCount: { $sum: 1 }
    }
  },
  {
    $project: {
      category: "$_id",
      priceRange: {
        min: "$minPrice",
        max: "$maxPrice",
        avg: { $round: ["$avgPrice", 2] },
        spread: { $subtract: ["$maxPrice", "$minPrice"] }
      },
      productCount: 1
    }
  },
  { $sort: { "priceRange.spread": -1 } }
])
```

## Summary

`$min` and `$max` accumulators are essential for range analysis, boundary detection, and comparative reporting in MongoDB aggregation. They work seamlessly with numeric values, dates, and string fields. Combined with `$avg` and `$sum` in the same `$group` stage, they provide a complete statistical summary in a single pipeline pass. In `$setWindowFields`, they enable per-document running minimums and maximums for time-series analysis.
