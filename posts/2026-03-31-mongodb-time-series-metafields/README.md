# How to Design MetaFields for Time Series Collections in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Time Series, MetaField, Schema Design, Performance

Description: Design metaField values for MongoDB time series collections to improve bucket efficiency, query performance, and storage compression.

---

When creating a time series collection in MongoDB, the `metaField` is an optional field that identifies the source or context of each measurement. MongoDB uses the metaField to group measurements together into the same internal bucket, which dramatically improves compression and query performance.

## What is the MetaField?

MongoDB stores time series measurements in internal "bucket" documents. Without a metaField, all measurements land in the same bucket regardless of source. With a metaField, measurements from the same source (e.g., the same sensor or the same host) are co-located in the same bucket.

```javascript
// Creating a time series collection with a metaField
db.createCollection("sensorReadings", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",   // the top-level field that identifies the source
    granularity: "seconds"
  }
});
```

## What Should Go in the MetaField?

The metaField should contain **static or slowly changing** attributes that identify the data source. It should NOT contain the measurement value.

**Good metaField candidates:**
- Sensor ID, device ID, machine ID
- Region, datacenter, hostname
- Application name, service name
- Product SKU, customer segment

**Bad metaField candidates:**
- Temperature, pressure, CPU usage (these are measurements)
- Timestamps (these are the timeField)
- Frequently changing status flags

## Flat vs. Nested MetaField Structure

```javascript
// Option 1: Flat metaField (simpler, fewer buckets)
{
  timestamp: ISODate("2026-03-31T10:00:00Z"),
  metadata: "sensor-42",
  temperature: 22.5
}

// Option 2: Nested metaField object (more dimensions, better filtering)
{
  timestamp: ISODate("2026-03-31T10:00:00Z"),
  metadata: {
    sensorId: "sensor-42",
    location: "warehouse-north",
    type: "temperature"
  },
  value: 22.5
}
```

A nested object is preferred when you need to query or group by multiple dimensions.

## Cardinality and Bucket Count

MongoDB creates one bucket per unique metaField value (within a time window). High cardinality metaFields create many buckets and reduce compression efficiency.

```text
Low cardinality (good):   10 sensors   -> 10 buckets per window
High cardinality (bad):   100,000 UUIDs -> 100,000 buckets per window
```

For high-cardinality identifiers, consider grouping them:

```javascript
// Instead of a raw UUID, use a derived identifier
metadata: {
  region: "us-east",
  deviceType: "thermostat",
  deviceId: "uuid-abc123"
}
```

Queries filtering on `metadata.region` will benefit from bucket co-location even with many device IDs.

## Indexing the MetaField

MongoDB automatically creates a compound index on `(metaField, timeField)`. You can add secondary indexes on subfields of the metaField:

```javascript
db.sensorReadings.createIndex({ "metadata.location": 1, "timestamp": 1 });
```

This supports queries like:

```javascript
db.sensorReadings.find({
  "metadata.location": "warehouse-north",
  timestamp: { $gte: ISODate("2026-03-31T00:00:00Z") }
});
```

## Avoiding Common MetaField Mistakes

```javascript
// WRONG: using a high-cardinality value as metaField top-level string
// This creates too many buckets
{ metadata: "uuid-abc123-def456", timestamp: ..., value: 23.1 }

// WRONG: including the measurement in the metaField
{ metadata: { sensorId: "s1", currentTemp: 22.5 }, timestamp: ... }

// RIGHT: metaField contains only static identifying attributes
{
  metadata: { sensorId: "s1", location: "room-A", type: "humidity" },
  timestamp: ISODate("2026-03-31T10:00:00Z"),
  humidity: 55.2
}
```

## Querying by MetaField

```javascript
// Find all readings from a specific sensor in the last hour
db.sensorReadings.find({
  "metadata.sensorId": "sensor-42",
  timestamp: {
    $gte: new Date(Date.now() - 3600 * 1000)
  }
}).sort({ timestamp: 1 });
```

## Summary

The metaField is the key to efficient time series storage in MongoDB. It should contain static identifiers that group related measurements together into the same internal bucket. Keep cardinality manageable, use a nested object for multi-dimensional filtering, and never include measurement values in the metaField. Properly designed metaFields lead to smaller bucket counts, better compression, and faster range queries.
