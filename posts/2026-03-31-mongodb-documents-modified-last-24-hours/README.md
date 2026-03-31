# How to Find Documents Modified in the Last 24 Hours in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Date, Filter, Change Tracking

Description: Learn how to find documents modified in the last 24 hours in MongoDB using date range queries on updatedAt fields and ObjectId timestamp extraction.

---

Finding recently modified documents is essential for change detection, audit trails, and incremental data sync. MongoDB supports several approaches depending on whether you maintain an `updatedAt` field or rely on ObjectId timestamps.

## Query by updatedAt Field

The most reliable approach is maintaining an `updatedAt` field on every document and querying with a date range:

```javascript
const twentyFourHoursAgo = new Date(Date.now() - 24 * 60 * 60 * 1000)

db.orders.find({
  updatedAt: { $gte: twentyFourHoursAgo }
}).sort({ updatedAt: -1 })
```

Count documents modified in the last 24 hours:

```javascript
const count = db.orders.countDocuments({
  updatedAt: { $gte: twentyFourHoursAgo }
})
print(`Modified in last 24h: ${count}`)
```

## Set updatedAt on Every Write

Ensure your application always sets `updatedAt` on updates:

```javascript
db.orders.updateOne(
  { _id: orderId },
  {
    $set: { status: "shipped", updatedAt: new Date() }
  }
)
```

For bulk updates:

```javascript
db.orders.updateMany(
  { status: "pending" },
  { $set: { status: "processing", updatedAt: new Date() } }
)
```

## Python Example

```python
from datetime import datetime, timezone, timedelta
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["salesdb"]

cutoff = datetime.now(timezone.utc) - timedelta(hours=24)

recently_modified = list(db.orders.find(
    { "updatedAt": { "$gte": cutoff } },
    { "_id": 1, "status": 1, "updatedAt": 1 }
).sort("updatedAt", -1))

print(f"Documents modified in last 24 hours: {len(recently_modified)}")
```

## Use ObjectId Timestamp for Creation Time

If you want documents CREATED in the last 24 hours, you can use the ObjectId's embedded timestamp without an `updatedAt` field:

```javascript
const twentyFourHoursAgo = new Date(Date.now() - 24 * 60 * 60 * 1000)
const minId = ObjectId.createFromTime(Math.floor(twentyFourHoursAgo.getTime() / 1000))

db.orders.find({
  _id: { $gte: minId }
})
```

This is efficient because `_id` is always indexed.

## Aggregation - Count by Hour for the Last 24 Hours

```javascript
const twentyFourHoursAgo = new Date(Date.now() - 24 * 60 * 60 * 1000)

db.orders.aggregate([
  { $match: { updatedAt: { $gte: twentyFourHoursAgo } } },
  {
    $group: {
      _id: {
        year: { $year: "$updatedAt" },
        month: { $month: "$updatedAt" },
        day: { $dayOfMonth: "$updatedAt" },
        hour: { $hour: "$updatedAt" }
      },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1, "_id.day": 1, "_id.hour": 1 } }
])
```

## Index the updatedAt Field

For performance on large collections, index `updatedAt`:

```javascript
db.orders.createIndex({ updatedAt: -1 })
```

Use a compound index if you typically filter by both status and updatedAt:

```javascript
db.orders.createIndex({ status: 1, updatedAt: -1 })
```

Verify the index is used:

```javascript
db.orders.find({ updatedAt: { $gte: twentyFourHoursAgo } })
  .explain("executionStats")
```

Look for `IXSCAN` in the winning plan and a low `totalDocsExamined` count.

## Summary

The most reliable way to find documents modified in the last 24 hours is to maintain an `updatedAt` field and query with `$gte: new Date(Date.now() - 86400000)`. Index `updatedAt` in descending order for efficient range queries. For documents created (not modified) in the last 24 hours, use the ObjectId timestamp approach as a zero-overhead alternative.
