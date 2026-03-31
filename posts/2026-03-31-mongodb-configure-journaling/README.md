# How to Configure Journaling in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Journaling, Durability, WiredTiger, Configuration

Description: Configure MongoDB journaling to ensure data durability and crash recovery, with options to balance performance against write safety.

---

## What Is MongoDB Journaling?

MongoDB uses a write-ahead log (WAL) called the journal to ensure data durability. When a write occurs, MongoDB first records the operation in the journal before applying it to data files. If MongoDB crashes, the journal allows recovery to a consistent state on restart.

WiredTiger uses its own journaling system separate from the MMAPv1 journal. By default, journaling is enabled and WiredTiger checkpoints every 60 seconds.

## Checking Journal Status

```javascript
db.adminCommand({ serverStatus: 1 }).dur
```

Or check the storage engine info:

```javascript
db.serverStatus().storageEngine
```

## Configuring Journal Commit Interval

The journal commit interval controls how often MongoDB flushes journal data to disk. Lower values improve durability but increase I/O.

In `mongod.conf`:

```yaml
storage:
  journal:
    enabled: true
    commitIntervalMs: 100
```

The default is 100ms. Valid range is 1-500ms.

## Disabling Journaling (Not Recommended for Production)

You can disable journaling on standalone instances for testing or specific workloads where you accept data loss risk:

```yaml
storage:
  journal:
    enabled: false
```

**Warning:** Disabling journaling means data written since the last checkpoint may be lost on crash. Never disable journaling on replica set members.

## WiredTiger Journal Settings

WiredTiger uses its own internal log. You can tune the log buffer size:

```yaml
storage:
  wiredTiger:
    engineConfig:
      journalCompressor: snappy
```

Available compressors: `none`, `snappy`, `zlib`, `zstd`.

Using compression reduces journal disk space at a slight CPU cost:

```bash
mongod --wiredTigerJournalCompressor snappy
```

## Journaling and Write Concern

Journal commit is tied to write concern. Using `j: true` waits for the operation to be written to the journal before acknowledging:

```javascript
db.orders.insertOne(
  { item: "widget", qty: 10 },
  { writeConcern: { w: 1, j: true } }
)
```

This ensures the write survives a MongoDB crash, even if a system-level crash occurs before the checkpoint.

## Monitoring Journal Activity

Check WiredTiger log stats:

```javascript
const wt = db.serverStatus().wiredTiger;
printjson({
  logBytesWritten: wt.log["log bytes written"],
  syncCalls: wt.log["log sync operations"],
  logFlushes: wt.log["log flush operations"]
});
```

## Summary

MongoDB journaling provides crash recovery by writing operations to a WAL before applying them to data files. Configure `commitIntervalMs` in `mongod.conf` to tune the flush interval, use `j: true` in write concerns for guaranteed durability, and choose an appropriate journal compressor to balance disk usage with CPU overhead. Keep journaling enabled on all production instances.
