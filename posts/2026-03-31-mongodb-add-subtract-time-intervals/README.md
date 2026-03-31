# How to Add and Subtract Time Intervals in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Date, Aggregation, Arithmetic, Query

Description: Learn how to add and subtract time intervals from dates in MongoDB using $dateAdd, $dateSubtract, and $add operators in queries and aggregations.

---

## Overview

Time arithmetic in MongoDB lets you compute expiry times, schedule future events, and filter documents relative to the current date. MongoDB 5.0 introduced `$dateAdd` and `$dateSubtract` for readable interval-based arithmetic.

## Using $dateAdd

`$dateAdd` adds a specified interval to a date.

```javascript
db.subscriptions.aggregate([
  {
    $project: {
      expiresAt: {
        $dateAdd: {
          startDate: "$startDate",
          unit: "day",
          amount: 30
        }
      }
    }
  }
]);
```

Supported units: `year`, `quarter`, `month`, `week`, `day`, `hour`, `minute`, `second`, `millisecond`.

## Using $dateSubtract

`$dateSubtract` works the same way but moves backward.

```javascript
db.events.aggregate([
  {
    $project: {
      windowStart: {
        $dateSubtract: {
          startDate: "$occurredAt",
          unit: "hour",
          amount: 24
        }
      }
    }
  }
]);
```

## Adding Timezone-Aware Intervals

Pass a `timezone` to handle daylight-saving-aware month/day boundaries.

```javascript
db.billing.aggregate([
  {
    $project: {
      nextBillingDate: {
        $dateAdd: {
          startDate: "$lastBilledAt",
          unit: "month",
          amount: 1,
          timezone: "America/Chicago"
        }
      }
    }
  }
]);
```

## Using $add for Millisecond Arithmetic (Pre-5.0)

Before MongoDB 5.0, you could add milliseconds directly.

```javascript
db.sessions.aggregate([
  {
    $project: {
      expiresAt: {
        $add: ["$createdAt", 3600000]  // add 1 hour in ms
      }
    }
  }
]);
```

## Filtering Documents Relative to Now

Compute a cutoff in the application layer and pass it to a query.

```javascript
const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);

db.logs.find({
  timestamp: { $gte: oneDayAgo }
});
```

Or compute entirely in the aggregation pipeline using `$$NOW`.

```javascript
db.sessions.aggregate([
  {
    $match: {
      $expr: {
        $lt: [
          "$expiresAt",
          {
            $dateAdd: {
              startDate: "$$NOW",
              unit: "hour",
              amount: 1
            }
          }
        ]
      }
    }
  }
]);
```

## Summary

MongoDB 5.0+ offers `$dateAdd` and `$dateSubtract` for readable, unit-based date arithmetic within aggregation pipelines. For older versions, use `$add` with millisecond values. Combine with `$$NOW` and `$expr` to filter documents dynamically relative to the current time.
