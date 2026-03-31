# How to Use New Features in MongoDB 5.0 (Time Series, Live Resharding)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Time Series, Resharding, Feature, Version

Description: Explore MongoDB 5.0's key new features including native time series collections and live resharding for sharded clusters.

---

## Introduction

MongoDB 5.0 introduced several important features: native time series collections, live resharding for sharded clusters, versioned API, window functions, and client-side field level encryption enhancements. This post covers the two most impactful for application developers and DBAs.

## Time Series Collections

Prior to MongoDB 5.0, storing time series data required manual bucketing patterns. MongoDB 5.0 introduces a dedicated time series collection type that automatically compresses and organizes sequential measurements.

### Creating a Time Series Collection

```javascript
db.createCollection("sensor_readings", {
  timeseries: {
    timeField: "timestamp",
    metaField: "sensorId",
    granularity: "minutes"
  },
  expireAfterSeconds: 2592000
})
```

- `timeField`: the field containing the measurement timestamp (required)
- `metaField`: a field that identifies the source (used for internal bucketing)
- `granularity`: hint for bucket sizing - `"seconds"`, `"minutes"`, or `"hours"`
- `expireAfterSeconds`: automatic TTL-based deletion

### Inserting Time Series Data

```javascript
db.sensor_readings.insertMany([
  {
    timestamp: ISODate("2025-11-01T10:00:00Z"),
    sensorId: "sensor_A",
    temperature: 22.5,
    humidity: 60
  },
  {
    timestamp: ISODate("2025-11-01T10:01:00Z"),
    sensorId: "sensor_A",
    temperature: 22.7,
    humidity: 61
  }
])
```

### Querying Time Series Data

Time series collections support all standard aggregation operations:

```javascript
db.sensor_readings.aggregate([
  {
    $match: {
      sensorId: "sensor_A",
      timestamp: {
        $gte: ISODate("2025-11-01T10:00:00Z"),
        $lt: ISODate("2025-11-01T11:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: { $dateTrunc: { date: "$timestamp", unit: "minute", binSize: 5 } },
      avgTemp: { $avg: "$temperature" }
    }
  }
])
```

Time series collections use an internal compressed bucket format, resulting in significant storage savings compared to regular collections for sequential measurements.

## Live Resharding

Before MongoDB 5.0, changing the shard key required taking the cluster offline and exporting/reimporting data. MongoDB 5.0 enables live resharding: changing the shard key while the cluster continues serving reads and writes.

### Resharding a Collection

```javascript
db.adminCommand({
  reshardCollection: "mydb.orders",
  key: { customerId: "hashed" }
})
```

MongoDB 5.0 begins copying documents to the new distribution in the background. The operation completes with a brief (sub-second) commit window where writes are blocked.

### Monitoring Resharding Progress

```javascript
db.getSiblingDB("admin").aggregate([
  { $currentOp: { allUsers: true } },
  { $match: { type: "op", "command.reshardCollection": { $exists: true } } }
])
```

Use `db.collection.getShardDistribution()` to verify the new distribution after completion.

## Window Functions in Aggregation

MongoDB 5.0 also adds `$setWindowFields` with window functions for moving averages, cumulative sums, and rank calculations:

```javascript
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { date: 1 },
      output: {
        runningTotal: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

## Summary

MongoDB 5.0's native time series collections provide automatic bucketing and compression for sequential measurement data, making them significantly more efficient than manual approaches. Live resharding eliminates the downtime previously required to change shard keys on growing collections. Window functions in `$setWindowFields` bring SQL-style analytics directly into MongoDB's aggregation pipeline.
