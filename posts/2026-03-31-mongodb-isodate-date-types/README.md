# How to Work with ISODate and Date Types in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, ISODate, Date, Aggregation

Description: Learn how to store, query, and manipulate dates in MongoDB using ISODate, the Date constructor, and aggregation date operators for time-based analysis.

---

## Date Storage in MongoDB

MongoDB stores dates as BSON `Date` objects - 64-bit integers representing milliseconds since the Unix epoch (UTC). The mongosh shell displays them as `ISODate("...")` strings. Always store dates as proper BSON Date objects, not strings, to enable date operators and range queries.

## Creating Dates

```javascript
// Current UTC time
const now = new Date();

// Specific date using ISO string
const d1 = new Date("2025-06-15T10:30:00Z");

// Using ISODate helper (same result)
const d2 = ISODate("2025-06-15T10:30:00Z");

// From milliseconds since epoch
const d3 = new Date(1718444400000);

db.events.insertOne({ name: "launch", ts: now });
```

## Querying by Date Range

```javascript
const start = new Date("2025-01-01T00:00:00Z");
const end   = new Date("2025-12-31T23:59:59.999Z");

db.orders.find({
  placedAt: { $gte: start, $lte: end }
});
```

## Sorting and Projecting Dates

```javascript
// Sort by most recent first
db.logs.find().sort({ timestamp: -1 }).limit(20);

// Project formatted date string
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      formattedDate: {
        $dateToString: { format: "%Y-%m-%d", date: "$placedAt" }
      }
    }
  }
]);
```

## Date Arithmetic in Aggregation

```javascript
// Add 30 days to a date
db.subscriptions.aggregate([
  {
    $addFields: {
      expiresAt: {
        $dateAdd: {
          startDate: "$startedAt",
          unit: "day",
          amount: 30
        }
      }
    }
  }
]);

// Calculate days between two dates
db.projects.aggregate([
  {
    $addFields: {
      durationDays: {
        $dateDiff: {
          startDate: "$startedAt",
          endDate: "$completedAt",
          unit: "day"
        }
      }
    }
  }
]);
```

## Extracting Date Parts

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: {
        year:  { $year: "$soldAt" },
        month: { $month: "$soldAt" },
        day:   { $dayOfMonth: "$soldAt" }
      },
      total: { $sum: "$amount" }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
]);
```

## Handling Timezones

MongoDB stores dates in UTC. Use the `timezone` option in aggregation operators for local time:

```javascript
db.bookings.aggregate([
  {
    $project: {
      localHour: {
        $hour: { date: "$createdAt", timezone: "America/New_York" }
      }
    }
  }
]);
```

## Common Pitfalls

Avoid storing dates as strings - they cannot be compared with date operators. If you have string dates, convert them:

```javascript
db.legacyOrders.find({ dateStr: { $exists: true } }).forEach(doc => {
  db.legacyOrders.updateOne(
    { _id: doc._id },
    { $set: { placedAt: new Date(doc.dateStr) }, $unset: { dateStr: "" } }
  );
});
```

## Summary

Always store dates as BSON Date objects using `new Date()` or `ISODate()` to enable range queries, sort operations, and aggregation date operators. Use `$dateAdd`, `$dateDiff`, and `$dateToString` for arithmetic and formatting, and specify the `timezone` option when displaying dates in local time.
