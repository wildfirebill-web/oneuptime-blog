# How to Use Compression with WiredTiger in MongoDB

Author: [OneUptime](https://www.github.com/oneuptime)

Tags: MongoDB, WiredTiger, Compression, Storage, Performance

Description: Learn how to configure WiredTiger compression for collections, indexes, and the journal in MongoDB to reduce storage footprint and improve I/O performance.

---

## Introduction

WiredTiger supports block compression for data files and the journal. Enabling compression reduces disk usage, which also reduces I/O because smaller files fit better in the filesystem cache. In most workloads, the CPU cost of compression is outweighed by the reduction in disk I/O. MongoDB supports snappy, zlib, and zstd compression algorithms.

## Compression Options

| Algorithm | Speed | Ratio | Best For |
|---|---|---|---|
| none | Fastest | 1:1 | CPU-bound workloads with fast NVMe |
| snappy | Fast | ~2:1 | Default, most workloads |
| zlib | Medium | ~3:1 | Storage-constrained environments |
| zstd | Fast | ~3.5:1 | MongoDB 4.2+, best general choice |

## Checking Current Compression Settings

```javascript
// View storage stats including compressed vs uncompressed size
db.orders.stats({ scale: 1048576 })
// Look for: storageSize (compressed) vs size (uncompressed)

// See compression ratio
var stats = db.orders.stats()
var ratio = stats.size / stats.storageSize
print("Compression ratio:", ratio.toFixed(2) + "x")
```

## Configuring Compression in mongod.conf

```yaml
storage:
  dbPath: /var/lib/mongodb
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4
      journalCompressor: zstd    # Journal compression

    collectionConfig:
      blockCompressor: zstd      # Default for all new collections

    indexConfig:
      prefixCompression: true    # Index key prefix compression (always recommended)
```

Restart mongod to apply:

```bash
sudo systemctl restart mongod
```

Note: Existing collections are not re-compressed automatically. Only new collections inherit the new default.

## Setting Compression Per Collection

Override the global default when creating a collection:

```javascript
// Create a collection with zstd compression
db.createCollection("logs", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"
    }
  }
})

// Create a collection with no compression (fast writes for time-series)
db.createCollection("telemetry_raw", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=none"
    }
  }
})

// Create a collection with zlib
db.createCollection("audit_archive", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zlib"
    }
  }
})
```

## Checking Per-Collection Compression Setting

```javascript
// View the storage config for a collection
db.getCollectionInfos({ name: "logs" })[0].options.storageEngine
```

## Re-Compressing Existing Collections

To change compression on an existing collection, you must rebuild it. Use `compact` to rewrite it in-place (WARNING: this locks the collection):

```javascript
// Compact rewrites the collection with the current blockCompressor setting
// First update the default in mongod.conf to the desired compressor, then:
db.runCommand({ compact: "orders" })
```

For zero-downtime re-compression on a replica set, use a rolling process:

1. Remove one secondary from the set (maintenance mode)
2. Export the data, drop the collection, recreate with new compression, reimport
3. Rejoin the replica set and let it resync
4. Repeat for each secondary, then step down the primary

```bash
# Export the collection
mongoexport --db mydb --collection orders --out orders_backup.json

# Drop and recreate with new compression
mongosh --eval '
  db.orders.drop()
  db.createCollection("orders", {
    storageEngine: { wiredTiger: { configString: "block_compressor=zstd" } }
  })
'

# Import
mongoimport --db mydb --collection orders --file orders_backup.json
```

## Index Compression

Prefix compression for indexes is controlled separately and is always recommended:

```javascript
// Create an index - prefix compression is used by default
db.orders.createIndex(
  { customerId: 1, createdAt: -1 },
  {
    storageEngine: {
      wiredTiger: { configString: "prefix_compression=true" }
    }
  }
)
```

## Network Compression (Client to Server)

Separate from storage compression, you can also compress the wire protocol between the driver and MongoDB:

```yaml
# In mongod.conf
net:
  compression:
    compressors: zstd,snappy,zlib   # Server-side supported compressors
```

In the connection string:

```javascript
// Node.js - request zstd wire compression
const client = new MongoClient(
  "mongodb://localhost:27017/?compressors=zstd"
)
```

## Measuring Compression Effectiveness

```javascript
// Before and after comparison
function compressionStats(collName) {
  var stats = db[collName].stats()
  return {
    collection: collName,
    uncompressedMB: (stats.size / 1048576).toFixed(2),
    compressedMB: (stats.storageSize / 1048576).toFixed(2),
    ratio: (stats.size / stats.storageSize).toFixed(2),
    indexSizeMB: (stats.totalIndexSize / 1048576).toFixed(2)
  }
}

printjson(compressionStats("orders"))
```

## Summary

WiredTiger compression in MongoDB reduces storage usage and often improves performance by reducing disk I/O. Configure the default compressor in `mongod.conf` under `storage.wiredTiger.collectionConfig.blockCompressor`. Use `zstd` for the best balance of speed and compression ratio on MongoDB 4.2+. Set compression per-collection at creation time using `configString`. Always enable index prefix compression with `prefixCompression: true`. To recompress existing collections, use `compact` or a rolling dump-and-reload procedure on replica set secondaries.
