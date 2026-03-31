# How to Optimize MongoDB for Time Series Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Time Series, Performance, Storage, IoT

Description: Learn how to use MongoDB time series collections, choose the right granularity and metaField, and design queries that take full advantage of time series optimizations.

---

MongoDB 5.0 introduced native time series collections with columnar compression, automatic bucketing, and specialized indexes that make them dramatically more efficient than regular collections for timestamped data like IoT readings, metrics, and events.

## Creating a Time Series Collection

```javascript
db.createCollection("sensor_readings", {
  timeseries: {
    timeField: "timestamp",     // field containing the date/time
    metaField: "metadata",      // field containing grouping metadata (e.g., sensorId)
    granularity: "seconds"      // "seconds", "minutes", or "hours"
  },
  expireAfterSeconds: 2592000   // optional: auto-expire after 30 days
})
```

## Inserting Data

Documents must include the `timeField` and optionally the `metaField`:

```javascript
db.sensor_readings.insertMany([
  {
    timestamp: new Date(),
    metadata: { sensorId: "sensor_42", location: "warehouse-A" },
    temperature: 22.5,
    humidity: 60
  },
  {
    timestamp: new Date(),
    metadata: { sensorId: "sensor_43", location: "warehouse-B" },
    temperature: 19.1,
    humidity: 55
  }
])
```

## Choosing Granularity

Granularity controls how MongoDB groups documents into internal buckets. Match it to your write frequency:

| Data Frequency         | Granularity |
|------------------------|------------|
| Multiple writes/second | `"seconds"` |
| Once per minute        | `"minutes"` |
| Once per hour or less  | `"hours"`   |

A mismatched granularity (e.g., `"hours"` for per-second data) creates overfull buckets and reduces compression benefits.

## Efficient Queries

### Range Query on timeField

```javascript
db.sensor_readings.find({
  "metadata.sensorId": "sensor_42",
  timestamp: {
    $gte: ISODate("2026-03-01"),
    $lt:  ISODate("2026-04-01")
  }
})
```

Time series collections automatically maintain a clustered index on `timeField`. Queries with time-range filters avoid full scans.

### Aggregation Over Time Windows

```javascript
db.sensor_readings.aggregate([
  {
    $match: {
      timestamp: { $gte: ISODate("2026-03-01") },
      "metadata.location": "warehouse-A"
    }
  },
  {
    $group: {
      _id: {
        hour: { $hour: "$timestamp" },
        day:  { $dayOfMonth: "$timestamp" }
      },
      avgTemp: { $avg: "$temperature" },
      maxTemp: { $max: "$temperature" }
    }
  }
])
```

## Adding Secondary Indexes

Create indexes on `metaField` sub-fields for fast filtering:

```javascript
db.sensor_readings.createIndex({ "metadata.sensorId": 1, timestamp: 1 })
```

Do not index individual measurement fields (temperature, humidity) unless they are used as query predicates.

## Storage Savings

Check compression ratio compared to a regular collection:

```javascript
db.sensor_readings.stats().storageSize    // compressed
db.sensor_readings.stats().size           // uncompressed logical size
```

Typical compression ratios are 10:1 to 40:1 for numeric measurement data.

## Automatic Deletion with TTL

The `expireAfterSeconds` option creates a TTL-like behavior that removes entire buckets, which is much more efficient than document-level TTL indexes on regular collections.

```javascript
// Change the TTL on an existing time series collection
db.runCommand({
  collMod: "sensor_readings",
  expireAfterSeconds: 86400   // 1 day
})
```

## Summary

Use MongoDB time series collections for timestamped workloads to benefit from columnar compression, automatic bucketing, and efficient time-range queries. Match `granularity` to your write frequency, keep the `metaField` for grouping identifiers, and add secondary indexes on metadata fields used in query predicates.
