# How to Use $merge to Sync Data Between Collections in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $merge, Sync, Collection

Description: Use MongoDB's $merge aggregation stage to synchronize data between collections, including denormalization, cross-collection joins, and read-optimized cache collections.

---

## Why Sync Collections with $merge

MongoDB's flexible document model often requires maintaining multiple representations of the same data:
- Normalized source collections for writes
- Denormalized collections for fast reads
- Enriched collections joining data from multiple sources
- Cache collections for expensive computed fields

`$merge` enables keeping these in sync incrementally without application-level code for every write path.

## Use Case 1: Denormalizing User Data into Orders

Keep a read-optimized orders collection with embedded user details:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "userInfo"
    }
  },
  {
    $set: {
      userName: { $arrayElemAt: ["$userInfo.name", 0] },
      userEmail: { $arrayElemAt: ["$userInfo.email", 0] },
      userTier: { $arrayElemAt: ["$userInfo.tier", 0] },
      syncedAt: "$$NOW"
    }
  },
  { $unset: "userInfo" },
  {
    $merge: {
      into: "orders_enriched",
      on: "_id",
      whenMatched: "merge",
      whenNotMatched: "insert"
    }
  }
])
```

## Use Case 2: Syncing Only Recently Updated Records

Use a checkpoint to sync only documents changed since the last run:

```javascript
async function syncChangedOrders() {
  // Read the last sync checkpoint
  const checkpoint = await db.collection("sync_checkpoints").findOne(
    { collection: "orders_enriched" }
  );

  const lastSyncAt = checkpoint?.lastSyncAt || new Date(0);
  const thisSyncAt = new Date();

  await db.collection("orders").aggregate([
    {
      $match: { updatedAt: { $gte: lastSyncAt } }
    },
    {
      $lookup: {
        from: "products",
        localField: "productId",
        foreignField: "_id",
        as: "product"
      }
    },
    {
      $set: {
        productName: { $arrayElemAt: ["$product.name", 0] },
        productCategory: { $arrayElemAt: ["$product.category", 0] }
      }
    },
    { $unset: "product" },
    {
      $merge: {
        into: "orders_enriched",
        on: "_id",
        whenMatched: "replace",
        whenNotMatched: "insert"
      }
    }
  ]).toArray();

  // Update checkpoint
  await db.collection("sync_checkpoints").updateOne(
    { collection: "orders_enriched" },
    { $set: { lastSyncAt: thisSyncAt } },
    { upsert: true }
  );
}
```

## Use Case 3: Cross-Database Sync

`$merge` can write to a different database:

```javascript
db.getSiblingDB("production").orders.aggregate([
  { $match: { status: "completed" } },
  {
    $merge: {
      into: { db: "analytics", coll: "completed_orders" },
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

## Use Case 4: Selective Field Sync

Sync only specific fields from the source to the target without touching others:

```javascript
db.products.aggregate([
  {
    $project: {
      price: 1,
      stockLevel: 1,
      lastModified: "$$NOW"
    }
  },
  {
    $merge: {
      into: "product_pricing_cache",
      on: "_id",
      whenMatched: "merge",       // Merge: only updates price, stockLevel, lastModified
      whenNotMatched: "insert"
    }
  }
])
```

## Handling Deletions

`$merge` cannot delete documents. Handle deletions separately:

```javascript
// Find documents in target that no longer exist in source
const sourceIds = await db.collection("users").distinct("_id");
await db.collection("user_cache").deleteMany({
  _id: { $nin: sourceIds }
});

// Then sync updates
await db.collection("users").aggregate([...syncPipeline]).toArray();
```

## Summary

`$merge` enables incremental collection synchronization by writing aggregation results - including `$lookup` joins - to a target collection with flexible conflict resolution. Use checkpoint-based matching to process only recently changed documents for efficiency. For cross-database sync, specify the target as `{ db: "otherDb", coll: "collection" }`. Handle deletions separately since `$merge` only inserts or updates; it cannot remove documents from the target that no longer exist in the source.
