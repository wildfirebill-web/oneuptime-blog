# How to Find and Remove Orphaned Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Orphaned Documents, $lookup, Data Integrity, Cleanup

Description: Learn how to find and remove orphaned documents in MongoDB - records that reference deleted parents - using $lookup, aggregation pipelines, and safe deletion strategies.

---

## Introduction

Orphaned documents are records that reference a parent or related document that no longer exists. Unlike relational databases with foreign key constraints, MongoDB does not enforce referential integrity natively. Orphans accumulate over time when parent records are deleted without cascading to related collections. Finding and removing them requires aggregation pipelines using `$lookup` to detect missing references.

## Understanding Orphaned Documents

```text
Example:
  - orders collection has { customerId: ObjectId("abc") }
  - customers collection no longer has a document with _id: ObjectId("abc")
  - The order is now "orphaned" - it references a non-existent customer
```

## Finding Orphaned Documents Using $lookup

Use a left outer join with `$lookup` and filter for unmatched documents:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  {
    $match: {
      customer: { $size: 0 } // No matching customer found
    }
  },
  {
    $project: {
      _id: 1,
      customerId: 1,
      orderDate: 1,
      amount: 1
    }
  }
]);
```

Documents with an empty `customer` array have a reference to a non-existent customer.

## Count Orphaned Documents Before Deleting

Always count first to understand the scope:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $match: { customer: { $size: 0 } } },
  { $count: "orphanedCount" }
]);
```

## Collect Orphaned IDs for Deletion

Collect the IDs of orphaned documents:

```javascript
const orphanedIds = db.orders
  .aggregate([
    {
      $lookup: {
        from: "customers",
        localField: "customerId",
        foreignField: "_id",
        as: "customer"
      }
    },
    { $match: { customer: { $size: 0 } } },
    { $project: { _id: 1 } }
  ])
  .map((doc) => doc._id);

print(`Found ${orphanedIds.length} orphaned orders`);
```

## Delete Orphaned Documents in Batches

Delete in batches to avoid large lock durations:

```javascript
const BATCH_SIZE = 500;
let totalDeleted = 0;

while (true) {
  const batch = db.orders
    .aggregate([
      {
        $lookup: {
          from: "customers",
          localField: "customerId",
          foreignField: "_id",
          as: "customer"
        }
      },
      { $match: { customer: { $size: 0 } } },
      { $project: { _id: 1 } },
      { $limit: BATCH_SIZE }
    ])
    .map((doc) => doc._id);

  if (batch.length === 0) break;

  const result = db.orders.deleteMany({ _id: { $in: batch } });
  totalDeleted += result.deletedCount;
  print(`Deleted ${result.deletedCount}, total: ${totalDeleted}`);
}

print(`Orphan cleanup complete. Total removed: ${totalDeleted}`);
```

## Finding Orphans with Null/Missing References

Also check for documents where the reference field is null or missing:

```javascript
db.orders.find({
  $or: [
    { customerId: null },
    { customerId: { $exists: false } }
  ]
});
```

## Handling Self-Referencing Orphans (Tree Structures)

For hierarchical data where nodes reference a parent:

```javascript
db.categories.aggregate([
  {
    $lookup: {
      from: "categories",
      localField: "parentId",
      foreignField: "_id",
      as: "parent"
    }
  },
  {
    $match: {
      parentId: { $ne: null },  // Has a parent reference
      parent: { $size: 0 }      // But parent doesn't exist
    }
  }
]);
```

## Python Script for Multi-Collection Orphan Cleanup

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["mydb"]

# Define parent-child collection pairs
relationships = [
    ("orders", "customerId", "customers"),
    ("orderItems", "orderId", "orders"),
    ("comments", "postId", "posts")
]

for child_col, ref_field, parent_col in relationships:
    pipeline = [
        {
            "$lookup": {
                "from": parent_col,
                "localField": ref_field,
                "foreignField": "_id",
                "as": "parent_doc"
            }
        },
        {"$match": {"parent_doc": {"$size": 0}}},
        {"$project": {"_id": 1}}
    ]

    orphan_ids = [doc["_id"] for doc in db[child_col].aggregate(pipeline)]
    if orphan_ids:
        result = db[child_col].delete_many({"_id": {"$in": orphan_ids}})
        print(f"Removed {result.deleted_count} orphans from {child_col}")
    else:
        print(f"No orphans in {child_col}")
```

## Preventing Orphans in Application Code

```javascript
// Cascade delete in application layer
async function deleteCustomer(customerId) {
  const session = client.startSession();
  session.startTransaction();

  try {
    // Delete all related records first
    await db.orders.deleteMany({ customerId }, { session });
    await db.reviews.deleteMany({ customerId }, { session });
    await db.customers.deleteOne({ _id: customerId }, { session });

    await session.commitTransaction();
  } catch (err) {
    await session.abortTransaction();
    throw err;
  } finally {
    session.endSession();
  }
}
```

## Summary

Finding MongoDB orphaned documents requires `$lookup` left joins that return empty arrays for unmatched references. Always count orphans before deleting, collect IDs using aggregation, and delete in batches to minimize impact on production workloads. Prevent future orphans by implementing cascade deletes in application code using multi-document transactions, and periodically run orphan audits as part of database maintenance.
