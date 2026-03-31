# How to Choose Between Collection Types in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Time Series, Capped Collection, Data Modeling

Description: Learn how to choose the right MongoDB collection type - regular, capped, time series, or clustered - based on your data access patterns and storage requirements.

---

## MongoDB Collection Types Overview

MongoDB offers four collection types, each optimized for different workloads. Choosing the wrong type leads to wasted storage, poor query performance, or unnecessary operational complexity.

```text
Regular       - General purpose, flexible, supports all features
Capped        - Fixed-size ring buffer, insertion order preserved, automatic expiry
Time Series   - Optimized for sequential time-stamped measurements
Clustered     - Documents physically ordered by a clustered index key
```

## Regular Collections

Use regular collections as your default choice. They support all MongoDB features, indexes, and query patterns. There are no restrictions on document size, insertion rate, or document removal.

```javascript
db.createCollection("orders");
// Suitable for: e-commerce orders, user profiles, content, transactions
```

Choose regular collections when:
- You need to update or delete arbitrary documents
- Data does not have a natural time series shape
- You need the full range of index types

## Capped Collections

Capped collections are fixed-size circular buffers. When the collection is full, the oldest documents are automatically overwritten. They maintain strict insertion order and support tailable cursors.

```javascript
db.createCollection("recentEvents", {
  capped: true,
  size: 52428800,   // 50 MB maximum size
  max: 100000       // Maximum 100,000 documents
});
// Suitable for: audit logs, recent activity feeds, message queues
```

Choose capped collections when:
- You only care about the most recent N documents
- You need to tail the collection for new data in real time
- You do not need to update or delete individual documents

## Time Series Collections

Time Series collections store measurements with timestamps in a compressed, columnar format. MongoDB automatically batches documents by time window and metadata field, yielding 10x storage savings for high-cardinality sensor or metric data.

```javascript
db.createCollection("sensorReadings", {
  timeseries: {
    timeField: "timestamp",
    metaField: "sensorId",
    granularity: "seconds"
  },
  expireAfterSeconds: 2592000   // 30 day TTL
});
// Suitable for: IoT sensors, application metrics, financial ticks
```

Choose time series collections when:
- Each document represents a time-stamped measurement
- You have many documents per device/sensor/source
- Storage efficiency and range-by-time queries are priorities

## Clustered Collections

Clustered collections store documents physically sorted by their `_id` field. This eliminates the separate `_id` index, saving storage and making range scans by `_id` faster.

```javascript
db.createCollection("invoices", {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true
  }
});
// Suitable for: time-ordered workloads, sequential access patterns
```

Choose clustered collections when:
- Your workload involves range scans by `_id`
- You generate monotonically increasing IDs (timestamps, sequences)
- You want to save storage by eliminating the separate `_id` index

## Quick Decision Guide

```text
Does data expire automatically?
  Yes, fixed size -> Capped
  Yes, time-based -> Time Series

Is data a series of measurements with timestamps?
  Yes, high volume -> Time Series

Do queries primarily scan by _id range?
  Yes -> Clustered

Everything else -> Regular
```

## Summary

MongoDB's four collection types serve distinct use cases. Regular collections handle the majority of workloads. Capped collections suit recent-data ring buffers with real-time tailing. Time Series collections dramatically reduce storage and improve range query performance for measurement data. Clustered collections optimize range scans on ordered `_id` fields. Evaluate your access patterns, expiry requirements, and document structure before choosing, as converting between some types requires data migration.
