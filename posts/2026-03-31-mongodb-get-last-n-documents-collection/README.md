# How to Get the Last N Documents from a Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Sort, Cursor, Pagination

Description: Learn multiple techniques to retrieve the last N documents from a MongoDB collection using sort, natural order, and aggregation pipelines.

---

Fetching the last N documents from a MongoDB collection is a frequent requirement - whether you need the most recent log entries, the latest orders, or the newest user signups. MongoDB provides several approaches depending on your data model and performance needs.

## Using Sort and Limit

The most straightforward approach is to sort in descending order and limit results:

```javascript
// Get the last 10 documents by _id (insertion order approximation)
db.events.find().sort({ _id: -1 }).limit(10)
```

Since ObjectId values are roughly time-ordered, sorting by `_id` descending gives you approximately the most recently inserted documents. For precise ordering, sort by a dedicated timestamp field:

```javascript
db.events.find().sort({ createdAt: -1 }).limit(10)
```

## Using Natural Order ($natural)

MongoDB's `$natural` hint sorts by the internal storage order. For uncapped collections, this is roughly insertion order:

```javascript
db.events.find().sort({ $natural: -1 }).limit(10)
```

For capped collections, `$natural` is very efficient because it uses the circular buffer structure directly. However, for regular collections, storage order can drift after updates and deletes, so a timestamp field is more reliable.

## Getting Last N with a Filter

Combine sort and limit with a filter condition:

```javascript
// Last 5 error events from a specific service
db.logs.find(
  { level: "error", service: "api-gateway" }
).sort({ timestamp: -1 }).limit(5)
```

## Using Aggregation Pipeline

The aggregation pipeline gives you more flexibility:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $sort: { placedAt: -1 } },
  { $limit: 10 },
  {
    $project: {
      orderId: 1,
      total: 1,
      placedAt: 1,
      _id: 0
    }
  }
])
```

## Reversing After Retrieval

If you want the last N documents but returned in chronological (ascending) order, sort descending to get them and then re-sort:

```javascript
db.messages.aggregate([
  { $sort: { sentAt: -1 } },
  { $limit: 20 },
  { $sort: { sentAt: 1 } }   // re-sort ascending for display
])
```

## Getting the Absolute Last Document

For a single most-recent document, `findOne` with descending sort is efficient:

```javascript
const latest = db.sensor_readings.findOne(
  { deviceId: "sensor-42" },
  {},
  { sort: { recordedAt: -1 } }
)
```

Or using the driver in Node.js:

```javascript
const latest = await db.collection("sensor_readings").findOne(
  { deviceId: "sensor-42" },
  { sort: { recordedAt: -1 } }
);
```

## Index Recommendations

Always back the sort field with an index to avoid a full collection scan:

```javascript
db.events.createIndex({ createdAt: -1 })

// Compound index for filtered queries
db.logs.createIndex({ service: 1, timestamp: -1 })
```

Without an index, MongoDB must scan the entire collection and sort in memory. Use `explain()` to verify:

```javascript
db.events.find().sort({ createdAt: -1 }).limit(10).explain("executionStats")
```

Look for `IXSCAN` in the winning plan. If you see `COLLSCAN`, add an index.

## Summary

To get the last N documents in MongoDB, sort by a timestamp or ObjectId field in descending order and apply `limit(N)`. Back the sort field with an index for performance. Use aggregation when you need filtering, projection, or re-ordering of results. For capped collections, `$natural: -1` is the most efficient approach.
