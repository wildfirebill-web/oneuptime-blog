# How to Enable and Configure Data Compression in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Compression, WiredTiger, Storage, Performance

Description: Learn how to configure WiredTiger compression algorithms in MongoDB to reduce disk usage and improve I/O performance for your collections and indexes.

---

## MongoDB Compression Overview

MongoDB uses WiredTiger, which supports three compression algorithms for collection data and indexes:

- **snappy** (default) - fast compression with moderate ratio, good for most workloads
- **zlib** - higher compression ratio, more CPU overhead, good for archival data
- **zstd** - best ratio with competitive speed, available from MongoDB 4.2+
- **none** - no compression, fastest writes but highest disk usage

Compression is configured at the collection level (for data) and globally (for indexes and the journal).

## Setting Compression in mongod.conf

Configure the default compression for all new collections and indexes:

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd
    indexConfig:
      prefixCompression: true
    engineConfig:
      journalCompressor: snappy
```

After changing this setting, restart `mongod`. Existing collections are not automatically recompressed - they retain their original settings.

## Setting Compression Per Collection

Override the global default when creating a collection:

```javascript
db.createCollection("archive_events", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"
    }
  }
});
```

Use `zstd` for collections with compressible text data or infrequent access, and `snappy` for hot collections where write latency matters.

## Checking Current Compression Settings

Inspect a collection's storage engine configuration:

```javascript
const info = db.getCollectionInfos({ name: "archive_events" })[0];
printjson(info.options.storageEngine);
```

Or use `collStats` to see actual compression savings:

```javascript
const s = db.archive_events.stats();
const ratio = (s.size / s.storageSize).toFixed(2);
print(`Logical size: ${s.size} bytes`);
print(`On-disk size: ${s.storageSize} bytes`);
print(`Compression ratio: ${ratio}x`);
```

## Recompressing an Existing Collection

To change compression for an existing collection, you must recreate it. The safest zero-downtime approach uses `$out` in an aggregation pipeline:

```javascript
// 1. Write to a new collection with desired compression
db.old_events.aggregate([
  { $match: {} },
  { $out: "new_events" }
]);

// 2. Create with correct compression first
db.createCollection("new_events", {
  storageEngine: { wiredTiger: { configString: "block_compressor=zstd" } }
});

// 3. Rename once migration is complete
db.old_events.renameCollection("old_events_backup");
db.new_events.renameCollection("old_events");
```

Alternatively, run `mongodump` and `mongorestore` with the new instance configured for the desired compressor.

## Compression for Index Prefix

Index prefix compression reduces index size on disk with no query performance impact:

```javascript
db.createCollection("users", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=snappy,prefix_compression=true"
    }
  }
});
```

Prefix compression is enabled by default for indexes and should not be disabled unless you have a very specific reason.

## Choosing the Right Compressor

```text
Workload Type          | Recommended Compressor
-----------------------|-----------------------
Hot write-heavy        | snappy
Balanced read/write    | snappy or zstd
Read-heavy / archival  | zstd
Legacy / low CPU       | zlib
Uncompressible data    | none
```

## Summary

Configure WiredTiger compression in `mongod.conf` for global defaults and per-collection using `createCollection` with a `configString`. Use `zstd` for the best compression ratio on MongoDB 4.2+, `snappy` for write-sensitive workloads, and `none` only for data that is already compressed (images, video). Check actual savings with `collStats` and recompress existing collections by recreating them with the new setting.
