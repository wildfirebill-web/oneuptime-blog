# How to Fix MongoError: WiredTiger Error - Cache Full in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, WiredTiger, Cache Full, Memory, Troubleshooting

Description: Learn how to diagnose and fix the WiredTiger cache full error in MongoDB by tuning cache size, reducing working set, and optimizing index and collection design.

---

## Overview

The WiredTiger storage engine maintains an in-memory cache for recently accessed data. When the cache reaches capacity and eviction cannot keep up with incoming writes, MongoDB logs:

```text
WT_PANIC: WiredTiger library panic
WiredTiger error (-31804) WT_CACHE_FULL
WiredTiger error (12) WT_ROLLBACK
```

This causes write stalls, transaction rollbacks, and in severe cases server crashes.

## Diagnosing the Problem

Check WiredTiger cache statistics:

```javascript
db.serverStatus().wiredTiger.cache
```

Key fields to watch:

```javascript
{
  "bytes currently in the cache": 4294967296,         // current cache usage
  "maximum bytes configured": 4294967296,              // cache limit
  "bytes read into cache": 10737418240,
  "bytes written from cache": 8589934592,
  "pages evicted by application threads": 1234,        // high = cache pressure
  "transaction checkpoints": 500,
  "cache overflow score": 75                            // anything over 10 is concerning
}
```

## Fix 1 - Increase WiredTiger Cache Size

By default, WiredTiger uses 50% of RAM minus 1 GB (minimum 256 MB). You can increase it:

### Via mongod.conf

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8   # set to ~50-80% of available RAM
```

### At runtime (no restart needed)

```javascript
db.adminCommand({
  setParameter: 1,
  wiredTigerEngineRuntimeConfig: "cache_size=8G"
});
```

### Choosing the right cache size

```text
Available RAM: 16 GB
OS and other processes: 4 GB
MongoDB cache: 10 GB (leave 2 GB buffer)
```

## Fix 2 - Reduce Working Set Size

If your working set is larger than available RAM, optimize to reduce it:

### Remove unused indexes

```javascript
// Find unused indexes
db.collection.aggregate([{ $indexStats: {} }]);

// Drop indexes with 0 accesses (after confirming they are truly unused)
db.orders.dropIndex("old_compound_index");
```

Indexes consume cache space. Every unused index wastes cache that could hold hot data.

### Use projections to reduce document size in cache

```javascript
// Without projection: entire document loaded into cache
db.users.find({ active: true });

// With projection: smaller documents, less cache pressure
db.users.find({ active: true }, { name: 1, email: 1, _id: 0 });
```

## Fix 3 - Enable Compression

WiredTiger uses Snappy compression by default. Switch to zstd for better compression ratios (MongoDB 4.2+):

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd
    indexConfig:
      prefixCompression: true
```

Note: Compression affects existing collections only on compaction.

## Fix 4 - Archive Old Data

If your collection has data that is rarely accessed, archive it to reduce the working set:

```javascript
// Move old records to an archive collection
const cutoff = new Date("2025-01-01");
const old = await db.collection("orders").find({ createdAt: { $lt: cutoff } }).toArray();

if (old.length > 0) {
  await db.collection("orders_archive").insertMany(old, { ordered: false });
  await db.collection("orders").deleteMany({ createdAt: { $lt: cutoff } });
}
```

## Fix 5 - Run a Compact Operation

Over time, collections accumulate fragmentation. Compact frees space and rewrites data:

```javascript
db.runCommand({ compact: "orders" });
```

`compact` blocks reads and writes on the collection - schedule during a maintenance window.

## Fix 6 - Monitor Cache Eviction

Set up monitoring to catch cache pressure early:

```javascript
function checkCacheHealth(db) {
  const cache = db.serverStatus().wiredTiger.cache;
  const usedBytes = cache["bytes currently in the cache"];
  const maxBytes = cache["maximum bytes configured"];
  const usagePercent = (usedBytes / maxBytes) * 100;

  console.log(`Cache usage: ${usagePercent.toFixed(1)}%`);

  if (usagePercent > 85) {
    console.warn("WARNING: WiredTiger cache above 85% - risk of cache pressure");
  }

  return usagePercent;
}
```

## Fix 7 - Scale Vertically or Horizontally

If the data is too large for the available memory:

- **Vertical scaling**: upgrade to a larger instance with more RAM.
- **Horizontal scaling**: shard the collection to distribute the working set across multiple nodes.

```javascript
// Enable sharding on a collection
sh.enableSharding("myapp");
sh.shardCollection("myapp.orders", { customerId: "hashed" });
```

## Summary

WiredTiger cache full errors indicate that your working set - the data MongoDB needs to service queries - exceeds the in-memory cache. The primary fixes are increasing the cache size to match available RAM, removing unused indexes, archiving infrequently accessed data, and enabling better compression. Monitor cache usage continuously and scale the instance or shard the collection if the working set cannot fit in memory even after optimization.
