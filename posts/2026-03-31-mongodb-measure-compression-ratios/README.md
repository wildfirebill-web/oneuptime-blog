# How to Measure Compression Ratios in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Compression, Storage, WiredTiger, Metric

Description: Learn how to measure actual compression ratios in MongoDB collections using collStats, serverStatus, and compare Snappy, Zlib, and Zstd performance.

---

## Why Measure Compression Ratios?

Choosing a compression algorithm is only half the battle - you need to verify that it actually compresses your data well. Document structure, field cardinality, and value entropy all affect real-world compression. Measuring ratios lets you validate your algorithm choice and quantify storage savings before committing to a configuration in production.

## Using collStats for Per-Collection Ratios

The primary tool for measuring compression is `db.collection.stats()`:

```javascript
const stats = db.orders.stats();
const ratio = (stats.size / stats.storageSize).toFixed(2);

print(`Collection: orders`);
print(`Logical size (uncompressed): ${(stats.size / 1e6).toFixed(2)} MB`);
print(`Storage size (compressed):   ${(stats.storageSize / 1e6).toFixed(2)} MB`);
print(`Compression ratio:            ${ratio}x`);
print(`Index size:                   ${(stats.totalIndexSize / 1e6).toFixed(2)} MB`);
```

Key fields:
- `size` - logical uncompressed size of all documents
- `storageSize` - actual on-disk size after compression
- `totalIndexSize` - disk space used by indexes (also compressed if prefix compression is on)

## Measuring All Collections at Once

```javascript
db.getCollectionNames().forEach((name) => {
  const s = db[name].stats();
  if (s.storageSize > 0) {
    const ratio = (s.size / s.storageSize).toFixed(2);
    print(`${name.padEnd(40)} ratio: ${ratio}x  storage: ${(s.storageSize / 1e6).toFixed(1)} MB`);
  }
});
```

## Comparing Algorithms with Sample Data

Create test collections with different compressors, load identical data, and compare:

```javascript
// Create collections with different compressors
db.createCollection("test_snappy", { storageEngine: { wiredTiger: { configString: "block_compressor=snappy" } } });
db.createCollection("test_zlib",   { storageEngine: { wiredTiger: { configString: "block_compressor=zlib" } } });
db.createCollection("test_zstd",   { storageEngine: { wiredTiger: { configString: "block_compressor=zstd" } } });

// Load 50,000 representative documents into each
const docs = Array.from({ length: 50000 }, (_, i) => ({
  orderId: `ORD-${i}`,
  customerId: `CUST-${Math.floor(Math.random() * 10000)}`,
  status: ["pending", "confirmed", "shipped", "delivered"][i % 4],
  items: Array.from({ length: Math.ceil(Math.random() * 5) }, (_, j) => ({
    sku: `SKU-${j}`, qty: j + 1, price: (j + 1) * 9.99
  })),
  createdAt: new Date()
}));

db.test_snappy.insertMany(docs);
db.test_zlib.insertMany(docs);
db.test_zstd.insertMany(docs);
```

Then compare:

```javascript
["test_snappy", "test_zlib", "test_zstd"].forEach((c) => {
  const s = db[c].stats();
  print(`${c}: ratio=${( s.size / s.storageSize).toFixed(2)}x, storage=${(s.storageSize/1e6).toFixed(1)}MB`);
});
```

Typical output for JSON-like documents:

```text
test_snappy: ratio=1.8x, storage=28.4MB
test_zlib:   ratio=4.2x, storage=12.1MB
test_zstd:   ratio=4.6x, storage=11.1MB
```

## WiredTiger Cache vs. Disk

Note that `storageSize` measures on-disk compressed size. In-memory (WiredTiger cache), documents are decompressed. Monitor cache efficiency separately:

```javascript
const engineStats = db.adminCommand({ serverStatus: 1 }).wiredTiger;
const cacheUsed = engineStats.cache["bytes currently in the cache"];
const cacheMax  = engineStats.cache["maximum bytes configured"];
print(`Cache utilization: ${((cacheUsed / cacheMax) * 100).toFixed(1)}%`);
```

High cache utilization with small on-disk size confirms compression is working and data fits in cache.

## Tracking Compression Savings Over Time

```bash
# Export collection stats to JSON for trending
mongosh --eval "
const s = db.orders.stats();
printjson({
  ts: new Date(),
  size: s.size,
  storageSize: s.storageSize,
  ratio: (s.size / s.storageSize).toFixed(2)
});
" >> /var/log/mongodb/compression_stats.jsonl
```

Run this via cron and feed the output to your observability platform.

## Summary

Measuring MongoDB compression ratios is straightforward with `collStats`. Compare `size` (logical) against `storageSize` (on-disk) for each collection. To choose between Snappy, Zlib, and Zstd, create test collections with identical sample data and compare their storage sizes directly. Zstd consistently delivers the best ratio for typical document workloads on MongoDB 4.2+, often 2-3x better than Snappy with minimal CPU overhead increase.
