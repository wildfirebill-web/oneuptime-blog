# How to Use $densify to Fill Time Gaps in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $densify, Time Series, Pipeline Stage, NoSQL

Description: Learn how to use MongoDB's $densify aggregation stage to fill gaps in time series data by generating synthetic documents for missing time intervals.

---

## What Is the $densify Stage?

The `$densify` stage (introduced in MongoDB 5.1) fills gaps in a sequence of documents by generating new documents at specified intervals. It is particularly useful for time series data where you need a continuous sequence even when no events occurred during certain periods.

```javascript
{
  $densify: {
    field: "fieldToFill",
    range: {
      step: stepValue,
      unit: "hour" | "day" | "week" | "month" | "quarter" | "year",
      bounds: "full" | "partition" | [lower, upper]
    },
    partitionByFields: ["groupByField1", "groupByField2"]  // optional
  }
}
```

## Basic Example - Daily Temperature Readings

Given sensor data with gaps:

```javascript
{ sensorId: "s1", timestamp: ISODate("2024-01-01"), temp: 22.5 }
{ sensorId: "s1", timestamp: ISODate("2024-01-03"), temp: 21.0 }
// January 2nd is missing
```

Fill the gap with `$densify`:

```javascript
db.sensorData.aggregate([
  {
    $densify: {
      field: "timestamp",
      range: {
        step: 1,
        unit: "day",
        bounds: [new Date("2024-01-01"), new Date("2024-01-04")]
      }
    }
  }
])
```

Output:

```javascript
{ sensorId: "s1", timestamp: ISODate("2024-01-01"), temp: 22.5 }
{ timestamp: ISODate("2024-01-02") }  // synthetic document, no temp value
{ sensorId: "s1", timestamp: ISODate("2024-01-03"), temp: 21.0 }
```

The synthetic document has only the `timestamp` field. Use `$fill` after `$densify` to populate missing values.

## Using bounds: "full"

The `"full"` option generates documents for the entire range from the minimum to maximum value of `field` in the collection:

```javascript
db.metrics.aggregate([
  {
    $densify: {
      field: "timestamp",
      range: {
        step: 1,
        unit: "hour",
        bounds: "full"
      }
    }
  }
])
```

## Using bounds: "partition"

With `partitionByFields`, `"partition"` fills gaps per partition from the min to max within each partition:

```javascript
db.sensorData.aggregate([
  {
    $densify: {
      field: "timestamp",
      partitionByFields: ["sensorId"],
      range: {
        step: 1,
        unit: "day",
        bounds: "partition"
      }
    }
  }
])
```

Each sensor gets its own densified time range independently.

## Densifying Multiple Sensors

```javascript
db.sensorData.aggregate([
  {
    $densify: {
      field: "timestamp",
      partitionByFields: ["sensorId", "location"],
      range: {
        step: 5,
        unit: "minute",
        bounds: [new Date("2024-01-01T00:00:00Z"), new Date("2024-01-02T00:00:00Z")]
      }
    }
  },
  // Fill missing values with the previous reading
  {
    $fill: {
      partitionByFields: ["sensorId", "location"],
      sortBy: { timestamp: 1 },
      output: {
        temp: { method: "locf" },
        humidity: { method: "locf" }
      }
    }
  }
])
```

## Numeric Densification

`$densify` also works with numeric fields (not just dates):

```javascript
db.scores.aggregate([
  {
    $densify: {
      field: "score",
      range: {
        step: 1,
        bounds: [0, 100]
      }
    }
  }
])
```

## Practical Use Case - Charting Metrics

When building dashboards, gaps in data cause broken charts. Use `$densify` followed by `$fill` to ensure smooth lines:

```javascript
db.pageViews.aggregate([
  { $match: { page: "/home", date: { $gte: new Date("2024-01-01") } } },
  {
    $densify: {
      field: "date",
      range: { step: 1, unit: "day", bounds: "full" }
    }
  },
  {
    $fill: {
      sortBy: { date: 1 },
      output: { views: { value: 0 } }  // use 0 for missing days
    }
  }
])
```

## Units Available

For date/time densification:

```text
millisecond, second, minute, hour, day, week, month, quarter, year
```

## Summary

The `$densify` stage solves the common problem of missing time intervals in time series data by generating synthetic placeholder documents at regular intervals. It works with dates and numbers, supports per-partition densification for grouped data, and integrates naturally with `$fill` to populate missing values. It is essential for building continuous time series charts and analytics dashboards.
