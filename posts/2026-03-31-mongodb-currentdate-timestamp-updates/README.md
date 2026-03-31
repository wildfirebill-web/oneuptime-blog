# How to Use $currentDate to Set Fields to the Current Timestamp in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Database, Update

Description: Learn how to use the $currentDate update operator in MongoDB to automatically set fields to the server's current date or timestamp on write.

---

MongoDB's `$currentDate` operator lets you set a field to the current date or timestamp at the moment an update runs - without needing to pass the date from your application code. This is useful for `updatedAt` tracking, audit logs, and expiry fields.

## Syntax

```javascript
db.collection.updateOne(
  { filter },
  { $currentDate: { fieldName: true } }
)
```

Setting the value to `true` stores a BSON `Date`. You can also use the `$type` sub-document to choose between `"date"` and `"timestamp"`:

```javascript
{ $currentDate: { fieldName: { $type: "timestamp" } } }
```

## Setting a Single Date Field

Mark a document's `updatedAt` field every time it is modified:

```javascript
db.orders.updateOne(
  { _id: ObjectId("64a1c2d3e4f5a6b7c8d9e0f1") },
  {
    $set: { status: "shipped" },
    $currentDate: { updatedAt: true }
  }
)
```

After this update, `updatedAt` contains the server's current UTC `Date`.

## Setting Multiple Date Fields at Once

You can populate several fields in a single operation:

```javascript
db.sessions.updateOne(
  { userId: "user_42" },
  {
    $currentDate: {
      lastAccessedAt: true,
      sessionRefreshedAt: { $type: "date" }
    }
  }
)
```

## Using a BSON Timestamp Instead of a Date

BSON Timestamps are 64-bit values used internally by MongoDB replication. You can use them for fields that need ordering guarantees across replica set members:

```javascript
db.events.updateOne(
  { _id: "evt_001" },
  {
    $currentDate: { processedAt: { $type: "timestamp" } }
  }
)
```

Note: For most application use cases, prefer `{ $type: "date" }` or `true`. BSON Timestamps are primarily useful for replication and oplog use cases.

## Combining with Upserts

`$currentDate` works seamlessly with upsert operations. When a document is inserted via upsert, the field is still set to the current server time:

```javascript
db.heartbeats.updateOne(
  { serviceId: "payments-svc" },
  {
    $set: { status: "alive" },
    $currentDate: { lastHeartbeat: true }
  },
  { upsert: true }
)
```

If the document does not exist, it is created with `lastHeartbeat` set to the time of the insert.

## Verifying the Result

```javascript
const doc = db.orders.findOne(
  { _id: ObjectId("64a1c2d3e4f5a6b7c8d9e0f1") },
  { updatedAt: 1 }
)
printjson(doc)
// { "_id": ..., "updatedAt": ISODate("2026-03-31T10:22:05.123Z") }
```

## Key Differences: $currentDate vs. Application-Side Timestamps

| Approach | Clock Used | Network Round-Trip |
|---|---|---|
| `$currentDate` | MongoDB server | Not needed |
| `new Date()` in app | Application server | Required |

Using `$currentDate` eliminates clock skew between application servers and ensures the timestamp reflects when the database write actually occurred.

## Summary

`$currentDate` sets document fields to the server's current date or BSON timestamp during an update. It removes the need to pass date values from the application layer, prevents clock skew across distributed services, and integrates cleanly with upserts. Use `true` or `{ $type: "date" }` for standard dates and `{ $type: "timestamp" }` for replication-aware ordering.
