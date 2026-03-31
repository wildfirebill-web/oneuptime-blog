# How to Copy a Collection Within the Same Database in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Database, Administration, Aggregation

Description: Learn how to copy a MongoDB collection within the same database using $out, aggregate with insertMany, or mongodump/mongorestore for data duplication tasks.

---

Copying a collection within the same database is useful for creating backups before schema migrations, testing queries on a snapshot, or duplicating configuration data. MongoDB does not have a single-command `COPY TABLE` equivalent, but several approaches achieve the same result.

## Method 1: Using $out in the Aggregation Pipeline

`$out` writes aggregation results to a new collection, replacing it if it already exists:

```javascript
db.originalCollection.aggregate([
  { $match: {} },
  { $out: "copyOfCollection" }
])
```

This is the fastest method for full copies. For a partial copy, filter with `$match`:

```javascript
db.orders.aggregate([
  { $match: { status: "completed", year: 2024 } },
  { $out: "orders_2024_backup" }
])
```

Important: `$out` replaces the target collection atomically. If the target exists, all existing documents are dropped first.

## Method 2: Using $merge for Incremental Copies

`$merge` (MongoDB 4.2+) inserts or updates documents into a target collection without fully replacing it:

```javascript
db.originalCollection.aggregate([
  { $match: {} },
  {
    $merge: {
      into: "copyOfCollection",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

This is useful when you want to keep an existing copy and refresh it with updated data.

## Method 3: Using find() + insertMany() in mongosh

For small to medium collections, a simple script works:

```javascript
// In mongosh
const docs = db.sourceCollection.find({}).toArray();
if (docs.length > 0) {
  db.targetCollection.insertMany(docs);
}
print(`Copied ${docs.length} documents`);
```

For large collections, batch the operation to avoid memory pressure:

```javascript
const batchSize = 1000;
let skip = 0;
let copied = 0;

while (true) {
  const batch = db.sourceCollection.find({}).skip(skip).limit(batchSize).toArray();
  if (batch.length === 0) break;
  db.targetCollection.insertMany(batch, { ordered: false });
  copied += batch.length;
  skip += batchSize;
}
print(`Total copied: ${copied}`);
```

## Method 4: Using mongodump and mongorestore

For a reliable copy that preserves indexes and metadata:

```bash
# Dump only the source collection
mongodump --db myDatabase --collection sourceCollection --out /tmp/dump

# Restore to a different collection name
mongorestore --db myDatabase \
  --collection targetCollection \
  /tmp/dump/myDatabase/sourceCollection.bson
```

## Copying Indexes Along with the Collection

Methods 1-3 do NOT copy indexes. After copying, recreate them:

```javascript
// List indexes on the source
const indexes = db.sourceCollection.getIndexes();

// Recreate each index on the target (skip the default _id index)
indexes
  .filter(idx => idx.name !== "_id_")
  .forEach(idx => {
    const keys = idx.key;
    const options = { name: idx.name };
    if (idx.unique) options.unique = true;
    if (idx.sparse) options.sparse = true;
    db.targetCollection.createIndex(keys, options);
  });
```

## Verifying the Copy

```javascript
const sourceCount  = db.sourceCollection.countDocuments();
const targetCount  = db.targetCollection.countDocuments();
print(`Source: ${sourceCount}, Target: ${targetCount}, Match: ${sourceCount === targetCount}`);
```

## Summary

Use `$out` for fast full copies within the same database, `$merge` for incremental updates to an existing copy, and `mongodump`/`mongorestore` when you also need indexes preserved. Always verify document counts after the copy and recreate indexes if you used the aggregation-based approaches.
