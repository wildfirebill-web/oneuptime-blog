# How to Work with Time Series Collections in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Time Series, IoT, Analytics, Performance

Description: Learn how to create and query MongoDB time series collections for efficient storage and analysis of time-stamped sensor and event data.

---

## What Are Time Series Collections?

MongoDB time series collections are purpose-built for storing time-stamped measurements. Introduced in MongoDB 5.0, they use columnar storage that groups documents by time bucket, reducing storage size and improving query performance for time-based workloads.

Time series collections are ideal for:
- IoT sensor data (temperature, humidity, pressure)
- Application metrics (CPU, memory, latency)
- Financial tick data
- User activity events

## Creating a Time Series Collection

```javascript
db.createCollection("sensorReadings", {
  timeseries: {
    timeField: "timestamp",    // required: field containing the date
    metaField: "metadata",     // optional: field for grouping/filtering
    granularity: "minutes"     // optional: seconds | minutes | hours
  },
  expireAfterSeconds: 86400 * 30  // optional: auto-delete after 30 days
})
```

The `timeField` must contain a BSON Date value. The `metaField` typically holds device or source identifiers.

## Inserting Time Series Data

```javascript
// Insert a single reading
db.sensorReadings.insertOne({
  metadata: { sensorId: "sensor-001", location: "warehouse-A", type: "temperature" },
  timestamp: new Date(),
  temperature: 22.5,
  humidity: 65.2
})

// Bulk insert readings
db.sensorReadings.insertMany([
  {
    metadata: { sensorId: "sensor-001", location: "warehouse-A" },
    timestamp: new Date("2024-06-01T10:00:00Z"),
    temperature: 21.8
  },
  {
    metadata: { sensorId: "sensor-002", location: "warehouse-B" },
    timestamp: new Date("2024-06-01T10:00:00Z"),
    temperature: 23.1
  }
])
```

## Querying Time Series Data

Standard queries work on time series collections:

```javascript
// Find readings from a specific sensor in a time range
db.sensorReadings.find({
  "metadata.sensorId": "sensor-001",
  timestamp: {
    $gte: ISODate("2024-06-01T00:00:00Z"),
    $lt: ISODate("2024-06-02T00:00:00Z")
  }
})

// Get latest reading per sensor
db.sensorReadings.aggregate([
  { $sort: { timestamp: -1 } },
  { $group: {
      _id: "$metadata.sensorId",
      latestReading: { $first: "$temperature" },
      lastSeen: { $first: "$timestamp" }
  }}
])
```

## Aggregating Time Series Data

Time series collections work seamlessly with aggregation pipelines:

```javascript
// Hourly averages for a sensor
db.sensorReadings.aggregate([
  { $match: {
      "metadata.sensorId": "sensor-001",
      timestamp: { $gte: ISODate("2024-06-01"), $lt: ISODate("2024-06-08") }
  }},
  { $group: {
      _id: {
        year: { $year: "$timestamp" },
        month: { $month: "$timestamp" },
        day: { $dayOfMonth: "$timestamp" },
        hour: { $hour: "$timestamp" }
      },
      avgTemp: { $avg: "$temperature" },
      minTemp: { $min: "$temperature" },
      maxTemp: { $max: "$temperature" },
      readingCount: { $sum: 1 }
  }},
  { $sort: { "_id.year": 1, "_id.month": 1, "_id.day": 1, "_id.hour": 1 } }
])
```

## Using $densify to Fill Gaps

When sensors miss readings, use `$densify` to fill time gaps:

```javascript
db.sensorReadings.aggregate([
  { $match: { "metadata.sensorId": "sensor-001" } },
  { $densify: {
      field: "timestamp",
      range: {
        step: 5,
        unit: "minute",
        bounds: [ISODate("2024-06-01T00:00:00Z"), ISODate("2024-06-01T06:00:00Z")]
      }
  }},
  { $fill: {
      output: { temperature: { method: "linear" } }
  }}
])
```

## Adding Secondary Indexes

```javascript
// Index on metadata field for filtering
db.sensorReadings.createIndex({ "metadata.sensorId": 1, timestamp: 1 })

// Index on a measurement field for threshold queries
db.sensorReadings.createIndex({ temperature: 1, timestamp: 1 })
```

## Setting Granularity

Choose granularity based on your data's typical ingestion frequency:

```javascript
// For sensors reporting every second
db.createCollection("highFreqReadings", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  }
})

// For hourly aggregated data
db.createCollection("hourlyMetrics", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "hours"
  }
})
```

Correct granularity significantly affects compression and bucket size.

## Checking Collection Info

```javascript
// View time series collection options
db.getCollectionInfos({ name: "sensorReadings" })

// Check storage statistics
db.sensorReadings.stats({ scale: 1024 })
```

## Summary

MongoDB time series collections provide automatic columnar bucketing, efficient compression, and optimized query performance for time-stamped data. Create them with a `timeField` and optional `metaField`, choose the right granularity for your ingestion rate, and use standard aggregation pipelines to compute averages, minimums, and maximums over time windows.
