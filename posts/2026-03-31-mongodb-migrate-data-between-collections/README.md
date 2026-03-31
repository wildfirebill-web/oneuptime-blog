# How to Migrate Data Between MongoDB Collections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Migration, Collection, Aggregation, Data Engineering

Description: Learn how to migrate and transform data between MongoDB collections using aggregation pipelines with $out and $merge, and Python scripts for complex transformations.

---

Migrating data between MongoDB collections is required when refactoring schemas, splitting or merging collections, or backfilling new fields. MongoDB provides built-in aggregation stages for bulk migrations, and Python scripts handle complex transformation logic.

## Using $out for Full Collection Replacement

The `$out` stage writes aggregation results to a new or existing collection, replacing any existing data:

```javascript
db.legacyUsers.aggregate([
  {
    $project: {
      _id: 1,
      username: "$name",
      email: 1,
      createdAt: "$created_date",
      isActive: { $ifNull: ["$active", true] }
    }
  },
  {
    $out: "users"
  }
])
```

This reads all documents from `legacyUsers`, transforms them, and writes them to the `users` collection, replacing any existing content.

## Using $merge for Upsert Migration

`$merge` gives more control - it can insert, replace, merge, or skip based on matching documents:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      totalWithTax: {
        $multiply: ["$total", 1.08]
      },
      migratedAt: "$$NOW"
    }
  },
  {
    $merge: {
      into: "orders_v2",
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

This upserts documents into `orders_v2` based on `_id`.

## Splitting One Collection into Two

Migrate different document types into separate collections:

```javascript
// Migrate products of type 'physical' to physicalProducts
db.products.aggregate([
  { $match: { type: "physical" } },
  { $unset: "type" },
  { $out: "physicalProducts" }
])

// Migrate digital products to digitalProducts
db.products.aggregate([
  { $match: { type: "digital" } },
  { $unset: "type" },
  { $out: "digitalProducts" }
])
```

## Python Script for Complex Transformations

For transformations that cannot be expressed in the aggregation pipeline:

```python
from pymongo import MongoClient
from bson import ObjectId
import re

client = MongoClient("mongodb://localhost:27017")
db = client["myapp"]

source = db["legacy_customers"]
target = db["customers"]

def transform_document(doc):
    # Split full_name into first_name and last_name
    name_parts = (doc.get("full_name") or "").split(" ", 1)
    transformed = {
        "_id": doc["_id"],
        "firstName": name_parts[0] if name_parts else "",
        "lastName": name_parts[1] if len(name_parts) > 1 else "",
        "email": (doc.get("email") or "").lower().strip(),
        "phone": re.sub(r"[^\d+]", "", doc.get("phone") or ""),
        "createdAt": doc.get("created_at") or doc.get("createdAt"),
    }
    return transformed

BATCH_SIZE = 1000
batch = []
total = 0
errors = 0

for doc in source.find({}):
    try:
        transformed = transform_document(doc)
        batch.append(transformed)
    except Exception as e:
        print(f"Transform error for {doc['_id']}: {e}")
        errors += 1

    if len(batch) >= BATCH_SIZE:
        target.insert_many(batch, ordered=False)
        total += len(batch)
        print(f"Migrated {total} documents")
        batch = []

if batch:
    target.insert_many(batch, ordered=False)
    total += len(batch)

print(f"Migration complete. {total} migrated, {errors} errors.")
client.close()
```

## Verifying the Migration

After migrating, verify row counts and spot-check records:

```javascript
// Compare counts
db.legacy_customers.countDocuments({})
db.customers.countDocuments({})

// Spot check a specific document
db.customers.findOne({ email: "alice@example.com" })

// Check for documents missing required fields
db.customers.countDocuments({ email: { $exists: false } })
```

## Summary

Migrating data between MongoDB collections is cleanest with `$out` for full collection rewrites and `$merge` for incremental upserts. Both stages support aggregation pipeline transformations before writing. For complex field-level transformations that cannot be expressed in MongoDB's expression language, use a Python script with batched `insert_many` calls. Always verify counts and spot-check data after migration before decommissioning the source collection.
