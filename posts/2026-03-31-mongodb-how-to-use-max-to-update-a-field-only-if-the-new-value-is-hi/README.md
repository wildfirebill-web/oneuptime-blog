# How to Use $max to Update a Field Only If the New Value Is Higher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update Operator, $max, Conditional Updates, NoSQL

Description: Learn how to use MongoDB's $max operator to update a field only when the new value is greater than the current value, perfect for tracking high-water marks and maximums.

---

## What Is the $max Operator?

The `$max` operator updates a field only if the specified value is greater than the current field value. If the field does not exist, `$max` sets it to the specified value. It is the counterpart to `$min` and uses BSON comparison order.

```javascript
db.collection.updateOne(
  { filter },
  { $max: { fieldName: value } }
)
```

## Basic Example

Track the highest score achieved by a player:

```javascript
// Current document: { playerId: "p1", highScore: 1500 }

db.players.updateOne(
  { playerId: "p1" },
  { $max: { highScore: 2000 } }
)
// highScore becomes 2000 (2000 > 1500)

db.players.updateOne(
  { playerId: "p1" },
  { $max: { highScore: 800 } }
)
// highScore stays 2000 (800 is NOT > 2000)
```

## Tracking Maximum Values

Record the peak concurrent user count for a service:

```javascript
db.services.updateOne(
  { serviceId: "api-gateway" },
  {
    $max: { peakConcurrentUsers: currentUsers },
    $set: { lastUpdatedAt: new Date() }
  }
)
```

## Field Does Not Exist

When the field does not exist, `$max` creates it:

```javascript
// Before: { _id: 1, productId: "p100" }
db.products.updateOne({ _id: 1 }, { $max: { maxQuantityOrdered: 50 } })
// After:  { _id: 1, productId: "p100", maxQuantityOrdered: 50 }
```

## Using $max with Dates

Track the most recent activity date:

```javascript
db.users.updateOne(
  { userId: "u42" },
  { $max: { lastActiveDate: new Date() } }
)
```

This only updates if the new date is later than the stored one - useful for eventually-consistent event streams where events may arrive out of order.

## Practical Use Case - Revenue Tracking

Track the single largest transaction per customer:

```javascript
function recordTransaction(customerId, amount) {
  return db.customers.updateOne(
    { customerId: customerId },
    {
      $max: { largestTransaction: amount },
      $inc: { totalTransactions: 1 }
    },
    { upsert: true }
  );
}
```

## Practical Use Case - Server Metrics

Track peak memory and CPU usage:

```javascript
function recordServerMetrics(serverId, cpuPercent, memoryMb) {
  return db.servers.updateOne(
    { serverId: serverId },
    {
      $max: { peakCpu: cpuPercent, peakMemoryMb: memoryMb },
      $set: { lastMetricAt: new Date() }
    },
    { upsert: true }
  );
}
```

## Using $max with updateMany

Establish high-water marks across an entire collection:

```javascript
// Set a minimum credit score floor of 300 for all accounts
db.accounts.updateMany(
  {},
  { $max: { creditScore: 300 } }
)
// For accounts with creditScore < 300, it will be set to 300
// For accounts with creditScore >= 300, no change
```

## Handling Out-of-Order Events

When processing events from distributed systems that may arrive out of order, `$max` ensures you always keep the latest known state:

```javascript
function processEvent(deviceId, timestamp, value) {
  return db.devices.updateOne(
    { deviceId: deviceId },
    {
      $max: {
        lastEventTimestamp: timestamp,
        peakValue: value
      }
    },
    { upsert: true }
  );
}
```

Even if an older event arrives after a newer one, the latest timestamp and peak value are preserved.

## $min vs $max Summary

```text
$min: Only updates if new value < current value (tracks minimums, floor values)
$max: Only updates if new value > current value (tracks maximums, high-water marks)
```

Both operators:
- Work with numbers, dates, and strings (BSON comparison order)
- Create the field if it doesn't exist
- Are atomic at the document level

## Summary

The `$max` operator enables safe, atomic high-water mark updates - it updates a field only when the new value exceeds the current one. This makes it invaluable for tracking peak metrics, personal bests, maximum transactions, and handling out-of-order event streams without race conditions.
