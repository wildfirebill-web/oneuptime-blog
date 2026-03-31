# How to Remove Duplicates from a MongoDB Collection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Duplicate, Aggregation, Data Cleanup, Index

Description: Learn how to find and remove duplicate documents from a MongoDB collection using aggregation to identify duplicates and bulk operations to delete them.

---

## Overview

Duplicate documents arise from missing unique constraints, failed idempotency logic, or data migration errors. MongoDB does not have a built-in deduplication command, but you can use the aggregation pipeline to identify duplicates and then delete the extra copies.

## Step 1: Identify Duplicate Documents

Use `$group` to find fields where more than one document shares the same value.

```javascript
db.users.aggregate([
  {
    $group: {
      _id: "$email",
      count: { $sum: 1 },
      ids: { $push: "$_id" }
    }
  },
  {
    $match: { count: { $gt: 1 } }
  }
]);
```

This returns each duplicate email along with an array of all `_id` values that share it.

## Step 2: Keep One, Delete the Rest

For each group, keep the first `_id` and delete the rest.

```javascript
db.users.aggregate([
  {
    $group: {
      _id: "$email",
      count: { $sum: 1 },
      ids: { $push: "$_id" }
    }
  },
  {
    $match: { count: { $gt: 1 } }
  }
]).forEach(function(doc) {
  doc.ids.shift(); // keep the first
  db.users.deleteMany({ _id: { $in: doc.ids } });
});
```

## Step 3: Verify No Duplicates Remain

```javascript
db.users.aggregate([
  {
    $group: {
      _id: "$email",
      count: { $sum: 1 }
    }
  },
  {
    $match: { count: { $gt: 1 } }
  }
]).toArray().length === 0;
```

## Deduplicating by Multiple Fields

If uniqueness is defined by a combination of fields, group by an object.

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { customerId: "$customerId", orderId: "$orderId" },
      count: { $sum: 1 },
      ids: { $push: "$_id" }
    }
  },
  { $match: { count: { $gt: 1 } } }
]).forEach(function(doc) {
  doc.ids.shift();
  db.orders.deleteMany({ _id: { $in: doc.ids } });
});
```

## Preventing Future Duplicates

After cleaning up, create a unique index to enforce uniqueness going forward.

```javascript
db.users.createIndex({ email: 1 }, { unique: true });
```

If duplicates still exist when you run this, the index creation will fail. Clean duplicates first.

## Using Bulk Operations for Large Collections

For very large collections, batch the deletes.

```javascript
const bulk = [];

db.users.aggregate([
  { $group: { _id: "$email", count: { $sum: 1 }, ids: { $push: "$_id" } } },
  { $match: { count: { $gt: 1 } } }
]).forEach(function(doc) {
  doc.ids.shift();
  doc.ids.forEach(id => {
    bulk.push({ deleteOne: { filter: { _id: id } } });
  });
});

if (bulk.length > 0) {
  db.users.bulkWrite(bulk, { ordered: false });
}
```

## Summary

To remove duplicates from a MongoDB collection, use `$group` and `$match` to identify documents sharing the same key fields, then delete all but one copy using `deleteMany` or bulk write operations. After cleanup, add a unique index to prevent duplicates from recurring.
