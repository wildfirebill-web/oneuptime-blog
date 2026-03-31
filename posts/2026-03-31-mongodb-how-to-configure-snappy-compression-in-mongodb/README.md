# How to Configure Snappy Compression in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Compression, Snappy, WiredTiger, Performance

Description: Learn how to configure Snappy compression in MongoDB's WiredTiger storage engine to reduce disk usage with minimal CPU overhead.

---

## Why Use Snappy Compression

Snappy is MongoDB's default compression algorithm for WiredTiger. It provides a good balance between compression ratio and CPU cost, making it suitable for most production workloads.

## Default Compression in MongoDB

MongoDB uses Snappy by default for the WiredTiger storage engine:

- Collection data: Snappy
- Journal: Snappy
- Index prefix compression is enabled by default

## Configuring Snappy via mongod.conf

To explicitly configure Snappy compression in `mongod.conf`:

```yaml
storage:
  dbPath: /var/lib/mongodb
  engine: wiredTiger
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
```

## Setting Compression via Command Line

Start `mongod` with Snappy compression:

```bash
mongod \
  --storageEngine wiredTiger \
  --wiredTigerCollectionBlockCompressor snappy \
  --dbpath /var/lib/mongodb
```

## Per-Collection Compression

Override the default at collection creation time:

```javascript
db.createCollection("logs", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=snappy"
    }
  }
})
```

## Checking Compression on an Existing Collection

```javascript
db.runCommand({ collStats: "logs", scale: 1048576 })
// Look for: wiredTiger.creationString containing "block_compressor=snappy"
```

## Comparing Compression Algorithms

| Algorithm | Speed   | Ratio  | CPU Cost |
|-----------|---------|--------|----------|
| Snappy    | Fast    | Medium | Low      |
| Zlib      | Medium  | High   | Medium   |
| Zstd      | Fast    | High   | Low-Med  |
| None      | Fastest | None   | None     |

## When to Choose Snappy

Use Snappy when:
- CPU is constrained and you cannot afford Zlib overhead
- Data is already partially compressed (images, blobs)
- Low-latency reads are more important than maximum space savings
- You want MongoDB's tested default behavior

## Verifying Compression Is Active

```javascript
db.runCommand({
  serverStatus: 1
})
// Check: wiredTiger.block-manager.blocks written
```

Or check storage ratio:

```javascript
const stats = db.myCollection.stats()
print("Compression ratio:", stats.size / stats.storageSize)
```

## Summary

Snappy is MongoDB's default WiredTiger compression algorithm, configured in `mongod.conf` under `storage.wiredTiger.collectionConfig.blockCompressor`. It offers fast decompression with moderate space savings, making it the right choice for CPU-sensitive workloads. Per-collection overrides allow mixing algorithms within the same deployment.
