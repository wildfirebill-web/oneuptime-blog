# How to Store and Query UTC Dates in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Date, Query

Description: Learn how to store dates in UTC and write accurate date range queries in MongoDB to avoid timezone bugs in your application.

---

MongoDB stores all dates internally as UTC milliseconds since the Unix epoch. This is the right default - it eliminates ambiguity and makes global applications predictable. However, getting dates in and out of MongoDB correctly requires understanding a few key patterns.

## Why UTC Storage Matters

When multiple users or servers in different timezones interact with your data, storing local times creates inconsistency. Two events that happened at the same moment will show different timestamps depending on where the document was created. UTC eliminates this by providing a single universal reference.

MongoDB's `Date` BSON type always stores as UTC internally. The challenge is ensuring your application code sends UTC-normalized values when inserting, and applies timezone offsets when displaying to users.

## Inserting UTC Dates

In the MongoDB shell, `new Date()` always creates a UTC date:

```javascript
db.events.insertOne({
  name: "User Signup",
  createdAt: new Date("2026-03-31T14:30:00Z")
});
```

The `Z` suffix explicitly denotes UTC. Without it, behavior depends on your environment. Always use the ISO 8601 format with the `Z` suffix or an explicit offset.

From a Node.js application using the native driver:

```javascript
const event = {
  name: "User Signup",
  createdAt: new Date() // JavaScript Date is UTC internally
};
await db.collection("events").insertOne(event);
```

From Python with PyMongo:

```python
from datetime import datetime, timezone

event = {
    "name": "User Signup",
    "createdAt": datetime.now(timezone.utc)
}
db.events.insert_one(event)
```

The `timezone.utc` argument is critical. Without it, Python's `datetime.now()` returns local time, which PyMongo will still store but without UTC normalization.

## Querying by Date Range

To find all events within a specific UTC date range:

```javascript
db.events.find({
  createdAt: {
    $gte: new Date("2026-03-01T00:00:00Z"),
    $lt:  new Date("2026-04-01T00:00:00Z")
  }
});
```

This returns all events in March 2026 UTC. The `$gte` and `$lt` operators work directly on BSON Date values.

## Indexing for Performance

Date range queries benefit enormously from an index on the date field:

```javascript
db.events.createIndex({ createdAt: 1 });
```

Without this index, MongoDB performs a full collection scan for every date range query. With a large collection, this becomes a serious bottleneck. Use `explain("executionStats")` to verify index usage:

```javascript
db.events.find({
  createdAt: { $gte: new Date("2026-03-01T00:00:00Z") }
}).explain("executionStats");
```

Look for `IXSCAN` in the winning plan and a low `totalDocsExamined` relative to `nReturned`.

## Converting for Display

When retrieving dates for display, apply the user's timezone offset in your application layer rather than storing multiple timezone representations:

```javascript
const doc = await db.collection("events").findOne({ name: "User Signup" });
const localTime = doc.createdAt.toLocaleString("en-US", {
  timeZone: "America/New_York"
});
```

For aggregation pipelines that need timezone-aware grouping, use the `$dateToParts` or `$dateToString` operators with a `timezone` argument:

```javascript
db.events.aggregate([
  {
    $project: {
      localDate: {
        $dateToString: {
          format: "%Y-%m-%d",
          date: "$createdAt",
          timezone: "America/New_York"
        }
      }
    }
  }
]);
```

## Summary

Always store dates as UTC in MongoDB using the BSON Date type. Use ISO 8601 strings with the `Z` suffix when inserting, ensure your driver sends UTC-normalized values, create indexes on date fields used in range queries, and handle timezone conversion at the display layer. This keeps your data consistent and your queries fast.
