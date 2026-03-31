# How to Create a Clustered Index in MongoDB 5.3+

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Clustered Index, Performance, Storage

Description: Learn how to create a clustered index in MongoDB 5.3+ to store collection documents ordered by a key, improving range scans and reducing storage overhead.

---

MongoDB 5.3 introduced clustered indexes, a storage-level feature that physically orders documents in the collection by the clustered key. This is different from regular indexes, which are separate B-tree structures pointing back to the data.

## What is a Clustered Index?

In a clustered collection:
- Documents are stored on disk in clustered key order
- The clustered key serves as the `_id` field
- Range scans on the clustered key avoid random I/O
- There is no separate index structure for the `_id` field - the collection IS the index

This is particularly valuable for time-series or sequential access patterns.

## Creating a Clustered Collection

A clustered index is specified at collection creation time and cannot be added to an existing collection:

```javascript
db.createCollection("sensorReadings", {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true,
    name: "clusterIdx"
  }
})
```

Currently, the clustered key must be `{ _id: 1 }`. MongoDB does not support clustering on arbitrary fields for general collections (time series collections handle this differently via `timeField`).

## Inserting Data

Insert data as usual. For time-ordered access patterns, use timestamps or sequential ObjectIDs as `_id`:

```javascript
db.sensorReadings.insertMany([
  { _id: new Date("2025-01-01T00:00:00Z"), sensorId: "s1", value: 23.4 },
  { _id: new Date("2025-01-01T00:01:00Z"), sensorId: "s1", value: 23.7 },
  { _id: new Date("2025-01-01T00:02:00Z"), sensorId: "s1", value: 23.5 }
])
```

## Querying with Range Scans

Range queries on `_id` in a clustered collection are highly efficient because documents are physically adjacent on disk:

```javascript
db.sensorReadings.find({
  _id: {
    $gte: new Date("2025-01-01T00:00:00Z"),
    $lt: new Date("2025-01-02T00:00:00Z")
  }
})
```

This query reads a contiguous block of storage rather than jumping around via a secondary index.

## Verifying Clustered Index

Check the collection's clustered index metadata:

```javascript
db.sensorReadings.getClusteredInfo()
```

Output:

```text
{
  "clusteredIndex": {
    "v": 2,
    "key": { "_id": 1 },
    "name": "clusterIdx",
    "unique": true
  }
}
```

Or check via `listCollections`:

```javascript
db.runCommand({ listCollections: 1, filter: { name: "sensorReadings" } })
```

## Adding Secondary Indexes

You can still create secondary indexes on clustered collections:

```javascript
db.sensorReadings.createIndex({ sensorId: 1, _id: 1 })
```

Secondary indexes on clustered collections store the clustered key as the record ID (instead of a RecordID pointer), making them slightly more compact.

## Clustered vs Regular Collections - When to Choose

| Scenario | Use Clustered |
|----------|--------------|
| Primary access is by `_id` range | Yes |
| Time-series / sequential inserts | Yes |
| Random `_id` lookups dominate | No (no benefit) |
| Need to cluster on non-`_id` field | Not supported |

## TTL with Clustered Collections

Clustered collections support TTL-like expiration natively via `expireAfterSeconds` without a separate TTL index:

```javascript
db.createCollection("sessionLogs", {
  clusteredIndex: { key: { _id: 1 }, unique: true },
  expireAfterSeconds: 3600
})
```

When `_id` is a date, MongoDB automatically expires documents older than the specified seconds.

## Summary

Clustered indexes in MongoDB 5.3+ co-locate documents on disk by the `_id` key, significantly improving range scan performance and reducing storage compared to separate index structures. They are best suited for time-ordered or sequential data accessed primarily by `_id` ranges. Combined with native TTL expiration, they provide an efficient alternative to time series collections for simpler use cases.
