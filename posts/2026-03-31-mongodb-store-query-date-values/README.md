# How to Store and Query Date Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Date, BSON, Query, Timezone

Description: Learn how MongoDB stores dates as BSON Date objects in UTC, how to query date ranges efficiently, and how to handle timezone conversions in aggregations.

---

## Date Type in BSON

MongoDB stores dates as BSON type `date` (type 9) - a 64-bit integer representing milliseconds since the Unix epoch (January 1, 1970 UTC). All dates are stored in UTC internally, regardless of the local timezone when the document was created.

## Inserting Date Values

```javascript
// Insert with the current UTC date/time
db.events.insertOne({
  name: "System Startup",
  createdAt: new Date(), // Current UTC time
  scheduledAt: new Date("2026-06-15T10:00:00Z"),
});
```

In Python with PyMongo, use `datetime` objects (always UTC):

```python
from datetime import datetime, timezone
import pymongo

client = pymongo.MongoClient("mongodb://localhost:27017")
db = client["myapp"]

db.events.insert_one({
    "name": "Deployment",
    "createdAt": datetime.now(timezone.utc),
    "scheduledAt": datetime(2026, 6, 15, 10, 0, 0, tzinfo=timezone.utc)
})
```

## Querying Date Ranges

```javascript
// Find events created in the last 7 days
const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
db.events.find({ createdAt: { $gte: sevenDaysAgo } });

// Find events within a specific date range
db.events.find({
  scheduledAt: {
    $gte: new Date("2026-06-01T00:00:00Z"),
    $lt: new Date("2026-07-01T00:00:00Z"),
  },
});
```

## Indexing Date Fields

Date fields are excellent candidates for indexes when used in range queries:

```javascript
db.events.createIndex({ createdAt: -1 }); // Descending for "most recent first" queries

// Compound index for filtered time-series queries
db.events.createIndex({ type: 1, createdAt: -1 });
```

## Aggregation with Date Operators

Extract date components for time-based grouping:

```javascript
db.events.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$createdAt" },
        month: { $month: "$createdAt" },
        day: { $dayOfMonth: "$createdAt" },
      },
      count: { $sum: 1 },
    },
  },
  { $sort: { "_id.year": 1, "_id.month": 1, "_id.day": 1 } },
]);
```

## Timezone Conversion in Aggregation

Since dates are stored in UTC, convert to local time using `$dateToParts` with a timezone:

```javascript
db.events.aggregate([
  {
    $project: {
      name: 1,
      localTime: {
        $dateToParts: {
          date: "$createdAt",
          timezone: "America/New_York",
        },
      },
    },
  },
]);
```

## Truncating Dates for Bucketing

Use `$dateTrunc` to bucket events by hour or day:

```javascript
db.events.aggregate([
  {
    $group: {
      _id: {
        $dateTrunc: {
          date: "$createdAt",
          unit: "hour",
          timezone: "UTC",
        },
      },
      count: { $sum: 1 },
    },
  },
]);
```

## Common Mistakes

Storing dates as strings is a frequent mistake that breaks sorting and range queries:

```javascript
// Wrong - stored as string, cannot use date range queries or date operators
db.events.insertOne({ name: "Bad Date", createdAt: "2026-03-31" });

// Find and fix string dates
db.events.find({ createdAt: { $type: "string" } }).forEach((doc) => {
  db.events.updateOne(
    { _id: doc._id },
    { $set: { createdAt: new Date(doc.createdAt) } }
  );
});
```

## Summary

MongoDB stores dates as BSON Date objects in UTC milliseconds since epoch. Always insert dates using `new Date()` or driver-native datetime types - never as strings. Query date ranges with `$gte`/`$lt` operators which use date indexes efficiently. Use aggregation date operators (`$year`, `$month`, `$dateTrunc`) for time-based grouping and `$dateToParts` with a timezone parameter to convert UTC to local time in pipelines.
