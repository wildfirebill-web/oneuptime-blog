# How to Use $avg as a Window Function in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Window Function, Aggregation, Analytics, Average

Description: Learn how to compute moving averages and rolling means using $avg as a window function in MongoDB's $setWindowFields stage.

---

The `$avg` window function in MongoDB's `$setWindowFields` stage computes averages over a sliding window of documents. Unlike `$group` which collapses rows, `$setWindowFields` attaches the computed average to each document, making it ideal for moving average calculations, trend smoothing, and benchmark comparisons.

## Simple Moving Average

Calculate a 7-day moving average of daily sales revenue per store:

```javascript
db.daily_sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$storeId",
      sortBy: { date: 1 },
      output: {
        movingAvg7Day: {
          $avg: "$revenue",
          window: {
            range: [-6, 0],
            unit: "day"
          }
        }
      }
    }
  },
  {
    $project: {
      storeId: 1,
      date: 1,
      revenue: 1,
      movingAvg7Day: { $round: ["$movingAvg7Day", 2] }
    }
  }
]);
```

`range: [-6, 0]` with `unit: "day"` includes the current day and the 6 preceding days.

## Document-Based Window Average

When data is not time-based, use a document count window instead:

```javascript
db.scores.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$playerId",
      sortBy: { gameNumber: 1 },
      output: {
        rollingAvg5Games: {
          $avg: "$score",
          window: {
            documents: [-4, 0]
          }
        }
      }
    }
  }
]);
```

`documents: [-4, 0]` includes the current document and the 4 documents before it in the sorted partition.

## Cumulative Average

Compute the running average from the first document to the current one:

```javascript
db.orders.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$salesRepId",
      sortBy: { orderDate: 1 },
      output: {
        cumulativeAvgOrderValue: {
          $avg: "$orderValue",
          window: {
            documents: ["unbounded", "current"]
          }
        }
      }
    }
  }
]);
```

This shows how each sales rep's average deal size has changed over time.

## Comparing Individual Values to Partition Average

Use `documents: ["unbounded", "unbounded"]` to compute the partition-wide average and attach it to every document for comparison:

```javascript
db.employees.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$department",
      sortBy: { salary: -1 },
      output: {
        departmentAvgSalary: {
          $avg: "$salary",
          window: {
            documents: ["unbounded", "unbounded"]
          }
        }
      }
    }
  },
  {
    $project: {
      name: 1,
      department: 1,
      salary: 1,
      departmentAvgSalary: { $round: ["$departmentAvgSalary", 0] },
      vsAverage: {
        $subtract: ["$salary", "$departmentAvgSalary"]
      }
    }
  }
]);
```

This identifies which employees earn above or below their department average.

## Centered Moving Average

Average over an equal window before and after the current document:

```javascript
db.sensor_readings.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$sensorId",
      sortBy: { recordedAt: 1 },
      output: {
        centeredAvg: {
          $avg: "$temperature",
          window: {
            documents: [-3, 3]   // 3 before + current + 3 after
          }
        }
      }
    }
  }
]);
```

Note: for forward-looking windows (`[0, N]` or `[-N, N]`), documents after the current one must be available, which means this works in batch aggregations but not in streaming contexts.

## Summary

The `$avg` window function enables moving averages, cumulative averages, and benchmark comparisons without collapsing the result set. Use range-based windows for time-series data, document-based windows for fixed-count rolling averages, and `["unbounded", "unbounded"]` partition windows when you need to compare each row against the group average. These patterns are foundational for analytics dashboards, anomaly detection, and trend analysis in MongoDB.
