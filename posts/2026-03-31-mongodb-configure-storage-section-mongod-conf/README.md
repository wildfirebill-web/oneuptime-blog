# How to Configure the storage Section in mongod.conf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Configuration, Storage, WiredTiger, Performance

Description: Learn how to configure the storage section in mongod.conf to set data paths, journal settings, and WiredTiger cache size for optimal MongoDB performance.

---

The `storage` section of `mongod.conf` determines where MongoDB writes data, how journaling works, and how much memory WiredTiger uses for its internal cache. Getting these settings right is fundamental to both performance and durability.

## Basic Structure

```yaml
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100
  directoryPerDB: false
  syncPeriodSecs: 60
  engine: wiredTiger
```

## Setting the Data Path

`dbPath` specifies the directory where MongoDB stores data files. The `mongod` user must own this directory.

```bash
sudo mkdir -p /data/mongodb
sudo chown -R mongodb:mongodb /data/mongodb
```

```yaml
storage:
  dbPath: /data/mongodb
```

Placing data on a dedicated volume separates I/O from the operating system and logs, which reduces contention on busy servers.

## Configuring the Journal

The journal ensures durability by logging write operations before applying them to data files. Keep journaling enabled in production.

```yaml
storage:
  journal:
    enabled: true
    commitIntervalMs: 100
```

`commitIntervalMs` controls how often the journal is flushed. The default is 100ms. Lowering this value reduces the window of potential data loss at the cost of additional I/O.

## Enabling directoryPerDB

When `directoryPerDB` is true, MongoDB creates a subdirectory for each database under `dbPath`. This simplifies moving or backing up individual databases.

```yaml
storage:
  directoryPerDB: true
```

## Configuring WiredTiger Cache Size

By default WiredTiger uses half of available RAM minus 1 GB. On a shared or memory-constrained server, set an explicit limit.

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2
```

Setting the cache too low forces frequent evictions and degrades read performance. Setting it too high leaves insufficient memory for the OS page cache and application processes.

## Setting syncPeriodSecs

`syncPeriodSecs` controls how often MongoDB flushes dirty pages from the WiredTiger cache to disk. The default is 60 seconds.

```yaml
storage:
  syncPeriodSecs: 60
```

This value interacts with checkpoint frequency. WiredTiger takes a checkpoint every 60 seconds by default, which is independent of this setting.

## Verifying Storage Configuration

After restarting `mongod`, confirm the active configuration.

```bash
mongosh --eval "db.adminCommand({ getCmdLineOpts: 1 }).parsed.storage"
```

Check the WiredTiger cache stats to see if the cache size is appropriate.

```javascript
db.serverStatus().wiredTiger.cache
```

## Summary

The `storage` section in `mongod.conf` controls the data directory, journal durability, per-database directory layout, and WiredTiger cache size. Always keep journaling enabled in production, place `dbPath` on a dedicated volume, and set `cacheSizeGB` explicitly on servers where MongoDB shares RAM with other processes.
