# How to Add a Field to All Documents in a Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Schema, Migration, Operator

Description: Learn how to add a new field to all existing documents in a MongoDB collection using updateMany with $set, and schema migration best practices.

---

When evolving your data model, you often need to add a new field to all existing documents. MongoDB's `updateMany` command with `$set` makes this a single operation. This is a common schema migration task in MongoDB since the schema is flexible but documents need to be explicitly updated.

## Adding a Field with updateMany

```javascript
// Add a "isVerified" field defaulting to false on all user documents
db.users.updateMany(
  {},  // empty filter matches ALL documents
  { $set: { isVerified: false } }
)
```

The empty filter `{}` targets every document in the collection.

## Adding a Field Only Where It Doesn't Exist

Use `$exists: false` to avoid overwriting documents that already have the field:

```javascript
db.users.updateMany(
  { isVerified: { $exists: false } },
  { $set: { isVerified: false } }
)
```

This is important for idempotent migrations - safe to run multiple times.

## Adding Multiple Fields at Once

```javascript
db.products.updateMany(
  {},
  {
    $set: {
      isActive: true,
      version: 1,
      metadata: {}
    }
  }
)
```

## Adding a Field with a Computed Value

For a field derived from existing data, use an aggregation pipeline update (MongoDB 4.2+):

```javascript
// Add "fullName" by concatenating firstName and lastName
db.users.updateMany(
  { fullName: { $exists: false } },
  [
    {
      $set: {
        fullName: {
          $concat: ["$firstName", " ", "$lastName"]
        }
      }
    }
  ]
)
```

## Adding a Timestamp Field

```javascript
// Add createdAt based on ObjectId timestamp (for existing documents)
db.orders.updateMany(
  { createdAt: { $exists: false } },
  [
    {
      $set: {
        createdAt: {
          $toDate: "$_id"
        }
      }
    }
  ]
)
```

## Batching Large Migrations

For very large collections, avoid timeouts by processing in batches:

```javascript
async function addFieldInBatches(collection, newField, defaultValue, batchSize = 1000) {
  let processed = 0;
  let hasMore = true;

  while (hasMore) {
    const docs = await collection.find(
      { [newField]: { $exists: false } },
      { projection: { _id: 1 } }
    ).limit(batchSize).toArray();

    if (docs.length === 0) {
      hasMore = false;
      break;
    }

    const ids = docs.map(d => d._id);
    const result = await collection.updateMany(
      { _id: { $in: ids } },
      { $set: { [newField]: defaultValue } }
    );

    processed += result.modifiedCount;
    console.log(`Processed ${processed} documents`);
  }

  return processed;
}
```

## Monitoring Migration Progress

```javascript
// Check how many documents still need the migration
const remaining = await db.collection("users").countDocuments({
  isVerified: { $exists: false }
});

console.log(`${remaining} documents still need migration`);
```

## Summary

Use `db.collection.updateMany({}, { $set: { newField: value } })` to add a field to all documents. Use `$exists: false` in the filter to make the migration idempotent. For computed field values based on existing data, use an aggregation pipeline update. For very large collections, process in batches to avoid long-running operations that may time out or impact production performance.
