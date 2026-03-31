# How to Query and Aggregate Time Series Data in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Time Series, Aggregation, Query, Analytics

Description: Query and aggregate MongoDB time series collections using date bucketing, window functions, and aggregation operators to compute metrics from sensor and event data.

---

MongoDB time series collections support the full aggregation pipeline, making them powerful for computing moving averages, daily rollups, percentile metrics, and other time-based analytics directly in the database.

## Basic Range Query

```javascript
// Find all readings from a specific sensor in the last 24 hours
db.sensorReadings.find({
  "metadata.sensorId": "sensor-42",
  timestamp: {
    $gte: new Date(Date.now() - 24 * 3600 * 1000)
  }
}).sort({ timestamp: 1 });
```

## Hourly Rollup with $dateToString

Group readings by hour to compute hourly averages:

```javascript
db.sensorReadings.aggregate([
  {
    $match: {
      "metadata.sensorId": "sensor-42",
      timestamp: { $gte: ISODate("2026-03-01T00:00:00Z") }
    }
  },
  {
    $group: {
      _id: {
        hour: { $dateToString: { format: "%Y-%m-%dT%H:00:00Z", date: "$timestamp" } },
        sensorId: "$metadata.sensorId"
      },
      avgTemp: { $avg: "$temperature" },
      maxTemp: { $max: "$temperature" },
      minTemp: { $min: "$temperature" },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id.hour": 1 } }
]);
```

## Using $dateTrunc for Time Bucketing

`$dateTrunc` is cleaner than `$dateToString` for bucketing:

```javascript
db.sensorReadings.aggregate([
  {
    $group: {
      _id: {
        bucket: { $dateTrunc: { date: "$timestamp", unit: "hour" } },
        sensor: "$metadata.sensorId"
      },
      avgTemp: { $avg: "$temperature" },
      readingCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.bucket": 1 } }
]);
```

## Moving Average with $setWindowFields

Compute a 5-reading moving average:

```javascript
db.sensorReadings.aggregate([
  { $match: { "metadata.sensorId": "sensor-42" } },
  { $sort: { timestamp: 1 } },
  {
    $setWindowFields: {
      partitionBy: "$metadata.sensorId",
      sortBy: { timestamp: 1 },
      output: {
        movingAvgTemp: {
          $avg: "$temperature",
          window: { documents: [-4, 0] }  // current + 4 preceding
        }
      }
    }
  }
]);
```

## Time-Based Window with $setWindowFields

```javascript
db.sensorReadings.aggregate([
  { $sort: { timestamp: 1 } },
  {
    $setWindowFields: {
      partitionBy: "$metadata.sensorId",
      sortBy: { timestamp: 1 },
      output: {
        rollingAvg1h: {
          $avg: "$temperature",
          window: {
            range: [-3600, 0],   // seconds relative to current document
            unit: "second"
          }
        }
      }
    }
  }
]);
```

## Daily Min/Max/Avg per Sensor

```javascript
db.sensorReadings.aggregate([
  {
    $match: {
      timestamp: {
        $gte: ISODate("2026-03-01T00:00:00Z"),
        $lt: ISODate("2026-04-01T00:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: {
        date: { $dateTrunc: { date: "$timestamp", unit: "day" } },
        sensor: "$metadata.sensorId"
      },
      avgTemp: { $avg: "$temperature" },
      maxTemp: { $max: "$temperature" },
      minTemp: { $min: "$temperature" }
    }
  },
  { $sort: { "_id.date": 1, "_id.sensor": 1 } }
]);
```

## Percentile Calculations

```javascript
db.sensorReadings.aggregate([
  { $match: { "metadata.sensorId": "sensor-42" } },
  {
    $group: {
      _id: null,
      p50: { $percentile: { input: "$temperature", p: [0.5], method: "approximate" } },
      p95: { $percentile: { input: "$temperature", p: [0.95], method: "approximate" } },
      p99: { $percentile: { input: "$temperature", p: [0.99], method: "approximate" } }
    }
  }
]);
```

## Query Performance Tips

```javascript
// Always include the timeField and metaField in $match to leverage the clustered index
db.sensorReadings.explain("executionStats").aggregate([
  {
    $match: {
      "metadata.sensorId": "sensor-42",  // hits metaField index
      timestamp: { $gte: ISODate("2026-03-31T00:00:00Z") }  // hits time-based clustering
    }
  }
]);
```

## Summary

MongoDB time series collections support efficient range queries and rich aggregation including hourly rollups with `$dateTrunc`, moving averages with `$setWindowFields`, and percentiles with `$percentile`. Always include both the metaField and timeField in `$match` stages to take advantage of the automatic clustered index and minimize the number of buckets scanned.
