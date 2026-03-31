# How to Configure Collection-Level Compression in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Compression, WiredTiger, Storage, Performance

Description: Learn how to configure collection-level compression in MongoDB using WiredTiger storage engine options to reduce disk usage and improve I/O performance.

---

## How MongoDB Compression Works

MongoDB's WiredTiger storage engine compresses collection data and indexes using pluggable compressors. Compression happens transparently at the storage layer - your application reads and writes documents normally while WiredTiger handles compression and decompression.

Available compressors:

- **snappy** (default) - Fast compression with moderate ratio, good for most workloads
- **zlib** - Higher compression ratio, slower CPU usage
- **zstd** - Best compression ratio with low CPU overhead (MongoDB 4.2+)
- **none** - No compression

## Setting Compression When Creating a Collection

Specify the compressor when creating a collection using `createCollection` with `storageEngine` options:

```javascript
db.createCollection("archive_logs", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"
    }
  }
})
```

To disable compression entirely:

```javascript
db.createCollection("high_frequency_cache", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=none"
    }
  }
})
```

## Setting Compression in a mongod Configuration File

Set the default compressor for all new collections in `mongod.conf`:

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd
    indexConfig:
      prefixCompression: true
```

Restart `mongod` after changing this setting. Existing collections retain their original compressor.

## Checking the Compressor on an Existing Collection

Use `db.collection.stats()` to view the current compressor:

```javascript
const stats = db.archive_logs.stats();
printjson(stats.wiredTiger["creationString"]);
```

Look for `block_compressor=` in the output string:

```
access_pattern_hint=none,allocation_size=4KB,app_metadata=(formatVersion=1),
block_compressor=zstd,...
```

## Changing Compression on an Existing Collection

WiredTiger does not support changing the compressor on an existing collection in place. To change compression, you must recreate the collection:

```javascript
// Step 1: Export data
// mongodump --db=mydb --collection=archive_logs --out=/tmp/backup

// Step 2: Drop the collection
db.archive_logs.drop()

// Step 3: Recreate with the new compressor
db.createCollection("archive_logs", {
  storageEngine: {
    wiredTiger: { configString: "block_compressor=zstd" }
  }
})

// Step 4: Restore data
// mongorestore --db=mydb --collection=archive_logs /tmp/backup/mydb/archive_logs.bson
```

## Choosing the Right Compressor

| Compressor | Speed | Compression Ratio | Best For |
|---|---|---|---|
| snappy | Fastest | Moderate | Hot data, OLTP |
| zlib | Slow | High | Archival data |
| zstd | Fast | Highest | General purpose (4.2+) |
| none | N/A | None | In-memory or SSD-heavy |

For most modern deployments on MongoDB 4.2+, `zstd` provides the best balance of compression ratio and CPU cost.

## Summary

MongoDB collection-level compression is configured via WiredTiger storage engine options when creating a collection or in the global `mongod.conf`. Choose `zstd` for the best space savings on modern MongoDB versions and `snappy` for workloads where decompression speed is critical. Changing compression on an existing collection requires a dump-drop-recreate cycle, so choose your compressor carefully at collection creation time.
