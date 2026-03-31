# How to Monitor Storage Engine Metrics in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, WiredTiger, Storage, Monitoring, Performance

Description: Learn how to monitor MongoDB WiredTiger storage engine metrics including cache utilization, checkpoint performance, and ticket availability using serverStatus.

---

## Why Monitor Storage Engine Metrics?

The WiredTiger storage engine is the core of MongoDB's data persistence layer. Its cache, checkpoint, and concurrency ticket systems directly impact query latency and write throughput. Monitoring these metrics helps you diagnose performance degradation, tune cache size, and identify I/O bottlenecks before they affect users.

## Accessing WiredTiger Metrics

All WiredTiger metrics are available under the `wiredTiger` key in `serverStatus`:

```javascript
const status = db.adminCommand({ serverStatus: 1 });
const wt = status.wiredTiger;
```

## Cache Metrics

The WiredTiger cache holds recently accessed data in memory. A well-tuned cache reduces disk I/O dramatically.

```javascript
const cache = db.adminCommand({ serverStatus: 1 }).wiredTiger.cache;

printjson({
  maxBytes:        cache["maximum bytes configured"],
  currentBytes:    cache["bytes currently in the cache"],
  dirtyBytes:      cache["tracked dirty bytes in the cache"],
  readIntoCache:   cache["pages read into cache"],
  writtenFromCache: cache["pages written from cache"],
  evictedModified: cache["modified pages evicted"],
  evictedUnmod:    cache["unmodified pages evicted"]
});
```

Key thresholds:
- Cache utilization above 90%: consider increasing `wiredTigerCacheSizeGB`
- High `modified pages evicted`: write workload is exceeding cache capacity
- High `pages read into cache`: working set exceeds cache size, causing disk I/O

## Concurrency Ticket Metrics

WiredTiger uses read and write tickets to limit concurrent operations. Running out of tickets causes queuing.

```javascript
const tickets = db.adminCommand({ serverStatus: 1 }).wiredTiger.concurrentTransactions;

printjson({
  readAvailable:    tickets.read.available,
  readOut:          tickets.read.out,
  readTotalTickets: tickets.read.totalTickets,
  writeAvailable:   tickets.write.available,
  writeOut:         tickets.write.out,
  writeTotalTickets: tickets.write.totalTickets
});
```

If `available` drops to 0, new operations must queue. Investigate slow queries using `db.currentOp()`.

## Checkpoint Metrics

WiredTiger checkpoints flush dirty cache pages to disk. Slow checkpoints cause write stalls.

```javascript
const checkpoints = db.adminCommand({ serverStatus: 1 }).wiredTiger.transaction;

printjson({
  checkpointsRun:    checkpoints["transaction checkpoint currently running"],
  checkpointTime:    checkpoints["transaction checkpoint total time (msecs)"],
  checkpointMax:     checkpoints["transaction checkpoint max time (msecs)"],
  checkpointMin:     checkpoints["transaction checkpoint min time (msecs)"]
});
```

Checkpoint time exceeding 60 seconds indicates I/O saturation. Consider moving the WiredTiger journal to a faster disk.

## Block Manager Metrics

```javascript
const block = db.adminCommand({ serverStatus: 1 }).wiredTiger["block-manager"];
printjson({
  bytesRead:    block["bytes read"],
  bytesWritten: block["bytes written"],
  mapReads:     block["mapped bytes read"]
});
```

Correlate `bytes read` and `bytes written` with system I/O metrics to identify disk throughput bottlenecks.

## Prometheus Monitoring

If you use the MongoDB Prometheus exporter, key WiredTiger metrics are exposed as:

```bash
mongodb_mongod_wiredtiger_cache_bytes{type="total"}
mongodb_mongod_wiredtiger_cache_bytes{type="dirty"}
mongodb_mongod_wiredtiger_concurrent_transactions_available_tickets{type="read"}
mongodb_mongod_wiredtiger_concurrent_transactions_available_tickets{type="write"}
```

A Prometheus alert for low write tickets:

```yaml
- alert: MongoDBLowWriteTickets
  expr: mongodb_mongod_wiredtiger_concurrent_transactions_available_tickets{type="write"} < 5
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "MongoDB WiredTiger write tickets critically low"
```

## Summary

Monitoring MongoDB's WiredTiger storage engine focuses on three areas: cache utilization (to ensure the working set fits in memory), concurrency tickets (to detect write/read queuing), and checkpoint performance (to catch I/O bottlenecks). These metrics are all accessible via `db.adminCommand({ serverStatus: 1 })` and can be exported to Prometheus for alerting. Regular monitoring of these metrics is essential for maintaining predictable MongoDB performance under load.
