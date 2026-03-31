# How to Configure Storage Engine Options in mongod.conf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, WiredTiger, Storage, Configuration, Performance

Description: Learn how to configure WiredTiger storage engine options in mongod.conf, including cache size, compression, journal, and checkpoint settings for MongoDB.

---

MongoDB has used WiredTiger as its default storage engine since version 3.2. While the defaults work for most deployments, tuning WiredTiger options in `mongod.conf` can significantly improve throughput and memory efficiency on production hardware.

## WiredTiger Configuration Hierarchy

Storage engine options live under `storage.wiredTiger` in `mongod.conf`.

```yaml
storage:
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4
      journalCompressor: snappy
      directoryForIndexes: false
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
```

## Setting Cache Size

WiredTiger's internal cache is the primary determinant of read performance. By default it uses `(RAM - 1 GB) / 2`. Set it explicitly to leave room for the OS page cache.

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4
```

On a 16 GB server running only MongoDB, a cache of 6-7 GB is a reasonable starting point. Monitor eviction pressure with `db.serverStatus().wiredTiger.cache`.

## Configuring Collection Compression

WiredTiger compresses data on disk. `snappy` is the default and offers a good balance of speed and compression ratio.

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy
```

Options are `none`, `snappy`, `zlib`, and `zstd`. `zlib` and `zstd` produce smaller files but require more CPU during reads and writes.

## Configuring Journal Compression

The WiredTiger journal can also be compressed.

```yaml
storage:
  wiredTiger:
    engineConfig:
      journalCompressor: snappy
```

Journal compression reduces I/O on write-heavy workloads with minimal CPU overhead when using `snappy`.

## Separating Indexes from Collections

Placing indexes in their own subdirectory under `dbPath` can improve I/O by distributing reads and writes across mount points.

```yaml
storage:
  wiredTiger:
    engineConfig:
      directoryForIndexes: true
```

When enabled, combine with OS-level symlinks or separate volumes to achieve I/O separation.

## Enabling Index Prefix Compression

Prefix compression reduces the on-disk size of B-tree index nodes by encoding shared key prefixes once per page.

```yaml
storage:
  wiredTiger:
    indexConfig:
      prefixCompression: true
```

This is enabled by default. Disable it only if you observe unexpected CPU overhead during index traversals, which is rare.

## Monitoring Storage Engine Health

```javascript
db.serverStatus().wiredTiger.cache
db.serverStatus().wiredTiger.transaction
db.stats()
```

The `cache` object shows bytes in cache, bytes read from disk, and eviction rates. High eviction rates indicate the cache is undersized.

## Summary

WiredTiger storage engine options in `mongod.conf` let you tune cache size, compression algorithm, journal compression, and index layout. Set `cacheSizeGB` explicitly on production servers, use `snappy` compression as a default, and enable `directoryForIndexes` when you can provide separate I/O paths for index and collection data. Monitor eviction rates in `db.serverStatus().wiredTiger.cache` to validate your settings.
