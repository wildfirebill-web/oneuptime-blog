# How to Use $densify to Fill Time Gaps in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Time Series

Description: Learn how MongoDB's $densify stage generates synthetic documents to fill missing time intervals in a dataset, enabling complete time series charts without gaps.

---

## What Is the $densify Stage?

The `$densify` stage generates new documents to fill in gaps in a sequence of values - most commonly time gaps in time series data. When you have measurements that only exist at some timestamps, `$densify` inserts placeholder documents at every specified interval so downstream stages always have a complete sequence to work with.

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $densify: {
      field: "<fieldToFill>",
      range: {
        step: <number>,
        unit: "<dateUnit>",   // for dates: "hour", "day", "week", etc.
        bounds: "full" | "partition" | [<lower>, <upper>]
      },
      partitionByFields: ["<groupingField>"]
    }
  }
])
```

## Example: Fill Hourly Temperature Readings

```javascript
// Data has readings at 6am, 9am, 3pm - missing 12pm
db.temperatures.aggregate([
  {
    $densify: {
      field: "timestamp",
      range: {
        step: 3,
        unit: "hour",
        bounds: "full"
      }
    }
  }
])
// Now includes a synthetic document at 12pm with timestamp only - other fields are missing
```

## Combining with $fill for Complete Records

`$densify` adds the missing timestamps; `$fill` then populates the missing values.

```javascript
db.metrics.aggregate([
  {
    $densify: {
      field: "ts",
      range: { step: 1, unit: "hour", bounds: "full" }
    }
  },
  {
    $fill: {
      output: {
        value: { method: "linear" }
      }
    }
  }
])
```

## Filling Gaps Between First and Last Document

Use `bounds: "full"` to fill from the earliest to the latest value in the dataset.

```javascript
db.dailyActiveUsers.aggregate([
  { $match: { appId: "app-1" } },
  {
    $densify: {
      field: "date",
      range: {
        step: 1,
        unit: "day",
        bounds: "full"
      }
    }
  }
])
```

## Filling Gaps with Explicit Bounds

Specify a fixed time window regardless of data extent.

```javascript
db.salesMetrics.aggregate([
  {
    $densify: {
      field: "reportDate",
      range: {
        step: 1,
        unit: "day",
        bounds: [
          new Date("2024-01-01"),
          new Date("2024-12-31")
        ]
      }
    }
  }
])
```

Every day in 2024 will have a document even if no sales occurred.

## Partition-Level Densification

Densify per group so each partition gets its own complete sequence.

```javascript
db.serverMetrics.aggregate([
  {
    $densify: {
      field: "timestamp",
      partitionByFields: ["serverId"],
      range: {
        step: 5,
        unit: "minute",
        bounds: "partition"
      }
    }
  }
])
// Each server has a complete set of 5-minute timestamps
```

With `bounds: "partition"`, each partition fills between its own first and last document.

## Non-Date Densification

`$densify` also works on numeric sequences.

```javascript
db.measurements.aggregate([
  {
    $densify: {
      field: "sensorReading",
      range: {
        step: 1,
        bounds: [0, 100]
      }
    }
  }
])
// Generates documents for every integer from 0 to 100
```

## Setting Values on Synthesized Documents

After densification, synthetic documents have the densified field set but other fields are absent. Use `$fill` or `$cond` to assign default values.

```javascript
db.metrics.aggregate([
  { $densify: { field: "ts", range: { step: 1, unit: "hour", bounds: "full" } } },
  {
    $addFields: {
      value: { $ifNull: ["$value", 0] }
    }
  }
])
```

## Summary

The `$densify` stage solves the missing-data problem in time series and ordered sequences by synthesizing placeholder documents at regular intervals. Combined with `$fill` for value interpolation, it enables smooth time series charts, SLA reporting, and any analytics that require complete, gap-free sequences.
