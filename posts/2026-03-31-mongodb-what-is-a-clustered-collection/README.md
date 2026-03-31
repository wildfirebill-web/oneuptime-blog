# What Is a Clustered Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Clustered Collection, Index, Performance, Time Series

Description: A clustered collection in MongoDB stores documents sorted by the clustered index key on disk, eliminating the need for a separate _id index and improving range scan performance.

---

## Overview

A clustered collection is a MongoDB collection where documents are physically stored sorted by the clustered index key rather than in insertion order. Unlike a standard collection where a separate B-tree `_id` index exists alongside the data, a clustered collection's index IS the data - the documents are stored in clustered index order, eliminating one level of indirection.

Clustered collections were introduced in MongoDB 5.3 and are particularly well-suited for time-series data, TTL workloads, and any access pattern that benefits from range scans on the primary key.

## Benefits of Clustered Collections

- **No separate _id index** - Documents and their index key are stored together, saving disk space
- **Faster range scans** - Documents with adjacent key values are physically adjacent on disk, improving I/O locality
- **Faster TTL deletions** - The TTL monitor can scan and delete expired documents sequentially rather than doing random I/O
- **Reduced storage** - Eliminates the overhead of the secondary `_id` index that all standard collections maintain

## Creating a Clustered Collection

```javascript
db.createCollection("sensorData", {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true,
    name: "sensorData clustered index"
  }
})
```

The clustered index key must be `{ _id: 1 }` - MongoDB requires the clustered index to be on the `_id` field. You can insert documents using any value as `_id`, including dates for time-series data.

## Using a Date as _id for Time Series

```javascript
// Insert sensor readings with timestamps as _id
db.sensorData.insertMany([
  { _id: new Date("2026-03-31T00:00:00Z"), sensorId: "s1", temp: 22.5 },
  { _id: new Date("2026-03-31T00:01:00Z"), sensorId: "s1", temp: 22.7 },
  { _id: new Date("2026-03-31T00:02:00Z"), sensorId: "s1", temp: 22.6 }
])

// Range query is very efficient - physically sequential read
db.sensorData.find({
  _id: {
    $gte: new Date("2026-03-31T00:00:00Z"),
    $lt: new Date("2026-03-31T01:00:00Z")
  }
})
```

## Combining with TTL

A clustered collection with a TTL index on `_id` is highly efficient for expiring time-series data:

```javascript
db.createCollection("events", {
  clusteredIndex: { key: { _id: 1 }, unique: true },
  expireAfterSeconds: 86400 // expire after 24 hours
})
```

Because documents are sorted by `_id` (the timestamp), the TTL monitor deletes expired documents with sequential reads and deletes rather than random I/O.

## Limitations

- The clustered index must be on `_id` only - you cannot cluster on another field.
- You cannot convert an existing standard collection to a clustered collection. Create the clustered collection first and migrate data.
- Additional secondary indexes can be added, but they are non-clustered.
- Clustered collections do not support all collection options (e.g., capped collections).

## Clustered Collection vs. Time Series Collection

MongoDB also has a dedicated time series collection type introduced in 5.0. Time series collections handle data from multiple sources (measurement series) and are optimized differently. Clustered collections are more general-purpose and give you direct control over the physical storage order.

## Checking if a Collection Is Clustered

```javascript
db.getCollectionInfos({ name: "sensorData" })
// Look for "clusteredIndex" in the options field
```

## Summary

Clustered collections store documents in `_id` key order on disk, eliminating the separate `_id` index and improving range scan I/O efficiency. They are best suited for time-series data, log ingestion, and TTL workloads where documents are naturally accessed by a time-ordered primary key.
