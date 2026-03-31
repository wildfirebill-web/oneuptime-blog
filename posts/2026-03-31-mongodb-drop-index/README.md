# How to Drop an Index in MongoDB with dropIndex()

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, dropIndex, Maintenance, Performance

Description: Learn how to drop indexes in MongoDB using dropIndex() and dropIndexes(), identify unused indexes with $indexStats, and safely remove indexes in production.

---

## Why Drop Indexes

Every index has a cost: it consumes RAM, increases disk usage, and slows down write operations (inserts, updates, and deletes). Dropping unused or redundant indexes:

- Reduces write overhead.
- Frees RAM for the working set.
- Simplifies maintenance and schema understanding.

You should periodically audit your indexes and drop any that are not used.

```mermaid
flowchart LR
    A[Identify unused indexes with $indexStats]
    A --> B[Confirm with explain() and profiler]
    B --> C[Drop index in off-peak window]
    C --> D[Monitor write and read performance]
```

## Syntax

```javascript
// Drop by key pattern
db.collection.dropIndex({ field: 1 })

// Drop by index name
db.collection.dropIndex("index_name")

// Drop all non-_id indexes
db.collection.dropIndexes()

// Drop multiple specific indexes (MongoDB 4.4+)
db.collection.dropIndexes(["index_name_1", "index_name_2"])
```

The `_id` index cannot be dropped.

## Examples

### Drop a Single Index by Key Pattern

```javascript
db.users.dropIndex({ email: 1 })
```

### Drop a Single Index by Name

```javascript
db.users.dropIndex("idx_email_unique")
```

### Drop All Non-_id Indexes

```javascript
db.users.dropIndexes()
```

Use with caution - this drops all indexes and queries will fall back to collection scans until indexes are rebuilt.

### Drop Multiple Indexes

```javascript
// MongoDB 4.4+
db.orders.dropIndexes(["idx_status", "idx_customer_status"])
```

### Identify Unused Indexes with $indexStats

Before dropping, check which indexes have not been used since the last mongod restart:

```javascript
db.orders.aggregate([{ $indexStats: {} }])
```

Sample output:

```javascript
[
  {
    "name": "idx_status",
    "key": { "status": 1 },
    "accesses": { "ops": 12847, "since": ISODate("2026-03-01T00:00:00Z") }
  },
  {
    "name": "idx_old_field",
    "key": { "legacyField": 1 },
    "accesses": { "ops": 0, "since": ISODate("2026-03-01T00:00:00Z") }
  }
]
```

An index with `ops: 0` has not been used since the server started - a strong candidate for removal.

### Node.js Example

```javascript
const { MongoClient } = require("mongodb");

async function auditAndDropIndexes() {
  const client = new MongoClient("mongodb://localhost:27017");
  await client.connect();

  const orders = client.db("shop").collection("orders");

  // Get index usage stats
  const stats = await orders.aggregate([{ $indexStats: {} }]).toArray();

  console.log("Index usage:");
  stats.forEach(s => {
    console.log(`  ${s.name}: ${s.accesses.ops} accesses since ${s.accesses.since}`);
  });

  // Find indexes with 0 accesses (excluding _id)
  const unusedIndexes = stats.filter(
    s => s.accesses.ops === 0 && s.name !== "_id_"
  );

  if (unusedIndexes.length === 0) {
    console.log("No unused indexes found.");
  } else {
    for (const idx of unusedIndexes) {
      console.log(`Dropping unused index: ${idx.name}`);
      await orders.dropIndex(idx.name);
    }
  }

  // List remaining indexes
  const remaining = await orders.indexes();
  console.log("Remaining indexes:", remaining.map(i => i.name));

  await client.close();
}

auditAndDropIndexes().catch(console.error);
```

### Check Current Indexes Before Dropping

Always inspect current indexes before dropping any:

```javascript
db.users.getIndexes()
```

Output:

```javascript
[
  { "v": 2, "key": { "_id": 1 }, "name": "_id_" },
  { "v": 2, "key": { "email": 1 }, "name": "email_1", "unique": true },
  { "v": 2, "key": { "username": 1 }, "name": "idx_username" }
]
```

### Handling Drop Errors

If you attempt to drop an index that does not exist, MongoDB returns an error:

```javascript
try {
  db.users.dropIndex("nonexistent_index")
} catch (err) {
  // MongoServerError: index not found with name [nonexistent_index]
  console.log("Error:", err.message)
}
```

### Rolling Index Removal in Replica Sets

In a replica set, dropping an index is replicated to all members. The drop propagates through the oplog and is applied to secondaries. Monitor replication lag during large index drops:

```javascript
// Check replication lag after dropping an index
rs.printReplicationInfo()
rs.printSecondaryReplicationInfo()
```

## Safe Drop Procedure

Follow these steps when dropping an index in production:

```text
1. Run $indexStats and identify the candidate index.
2. Verify with explain() on critical queries to confirm the index is not used.
3. Optionally use a MongoDB profiler window to observe real queries.
4. Pick an off-peak maintenance window.
5. Drop the index.
6. Monitor query performance and write throughput for 24-48 hours.
7. If queries degrade, re-create the index.
```

## Best Practices

- **Never drop an index without verifying it is unused.** Run `$indexStats` and review `explain()` output.
- **Drop in off-peak hours.** On large collections, dropping and re-creating an index blocks reads briefly at the start.
- **Keep the index definition documented** before dropping, so you can re-create it quickly if needed.
- **Be careful with `dropIndexes()` (no arguments)** - it drops all non-`_id` indexes and can cause immediate performance degradation.
- **Use index names** rather than key patterns when dropping to avoid ambiguity with compound indexes.
- **Monitor writes after dropping.** Removing an index should improve write performance; a degradation suggests the index had additional uses.

## Summary

`dropIndex()` removes a specific index from a MongoDB collection by key pattern or name. Use `dropIndexes()` to remove all non-`_id` indexes. Before dropping, use `$indexStats` to identify unused indexes (those with `ops: 0`), verify with `explain()` that critical queries do not rely on the index, and drop during off-peak hours. Monitor performance for 24-48 hours afterward to confirm the removal had no negative impact.
