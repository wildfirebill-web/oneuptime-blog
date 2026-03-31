# How to Set Up WiredTiger Cache Size in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, WiredTiger, Cache, Performance, Configuration

Description: Configure the WiredTiger cache size in MongoDB to optimize memory usage and improve read/write performance for your workload.

---

## Understanding WiredTiger Cache

MongoDB uses the WiredTiger storage engine by default. WiredTiger maintains an internal cache of frequently accessed data in memory. Properly sizing this cache is critical: too small causes frequent disk reads, too large leaves no memory for the OS page cache or other processes.

## Default Cache Size Formula

By default, MongoDB sets the WiredTiger cache to:

```text
max(50% of (RAM - 1GB), 256 MB)
```

For a server with 8 GB RAM: `max((8-1)*0.5, 0.25)` = 3.5 GB.

## How to Set Cache Size

### Via mongod.conf (Recommended)

```yaml
storage:
  dbPath: /var/lib/mongodb
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4
```

Apply the change:

```bash
sudo systemctl restart mongod
```

### Via Command Line

```bash
mongod --wiredTigerCacheSizeGB 4 --dbpath /var/lib/mongodb
```

### Dynamic Change (MongoDB 3.2+)

You can adjust the cache size without restarting using `setParameter`:

```javascript
db.adminCommand({ setParameter: 1, wiredTigerEngineRuntimeConfig: "cache_size=4G" })
```

## Verifying the Configuration

Check the configured and current cache size:

```javascript
db.serverStatus().wiredTiger.cache["maximum bytes configured"]
```

This returns the value in bytes. Divide by 1073741824 to get GB.

```javascript
const cacheBytes = db.serverStatus().wiredTiger.cache["maximum bytes configured"];
print("Cache size (GB):", cacheBytes / 1073741824);
```

## Cache Size Recommendations by Workload

| Workload Type | Recommended Cache Size |
|---|---|
| Read-heavy (hot dataset fits in RAM) | 60-70% of RAM |
| Mixed read/write | 50% of RAM |
| Write-heavy with large data sets | 40% of RAM |
| Containerized with 4 GB limit | 1.5 GB |

## Monitoring Cache Pressure

Watch for these indicators in `db.serverStatus().wiredTiger.cache`:

```javascript
const cache = db.serverStatus().wiredTiger.cache;
printjson({
  currentGB: cache["bytes currently in the cache"] / 1073741824,
  maxGB: cache["maximum bytes configured"] / 1073741824,
  dirtyPct: (cache["tracked dirty bytes in the cache"] / cache["maximum bytes configured"] * 100).toFixed(2) + "%",
  evictions: cache["pages evicted by application threads"]
});
```

High `pages evicted by application threads` means MongoDB is struggling to free cache under pressure - consider increasing cache size.

## Summary

Setting the WiredTiger cache size with `wiredTigerCacheSizeGB` in `mongod.conf` is the most reliable approach. Size it at roughly 50% of available RAM, leaving room for the OS page cache. Use `db.serverStatus().wiredTiger.cache` to monitor utilization and adjust dynamically using `setParameter` if needed without a restart.
