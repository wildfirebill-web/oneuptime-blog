# How WiredTiger Checkpoints Affect Write Performance in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, WiredTiger, Performance, Checkpoint, Storage Engine

Description: Learn how WiredTiger checkpoints work in MongoDB, how they affect write latency, and how to tune checkpoint frequency for your workload.

---

WiredTiger periodically writes a consistent snapshot of all in-memory data to disk through a process called checkpointing. While essential for durability and recovery, checkpoints can cause write latency spikes if not properly understood and tuned.

## What Is a Checkpoint?

A checkpoint is a point-in-time snapshot of all data pages written to the data files on disk. It provides a consistent recovery point. After a checkpoint, WiredTiger can recover from that snapshot rather than replaying the entire write-ahead journal from scratch.

By default, MongoDB triggers a checkpoint:
- Every 60 seconds, or
- When the journal file reaches 2 GB

## How Checkpoints Affect Writes

During a checkpoint, WiredTiger must flush all dirty pages to disk. This creates I/O pressure. If your storage subsystem is slow (spinning disks, saturated I/O), writes queued during a checkpoint will experience elevated latency.

You can observe checkpoint activity:

```javascript
const stats = db.serverStatus().wiredTiger;
console.log({
  checkpointsTotal:  stats["transaction"]["transaction checkpoints"],
  checkpointMaxMs:   stats["transaction"]["transaction checkpoint max time (msecs)"],
  checkpointMostRecent: stats["transaction"]["transaction checkpoint most recent time (msecs)"]
});
```

## Tuning Checkpoint Frequency

Increase the checkpoint interval to reduce I/O frequency (at the cost of longer recovery time after a crash):

```javascript
db.adminCommand({
  setParameter: 1,
  wiredTigerEngineRuntimeConfig: "checkpoint=(wait=120,log_size=0)"
});
```

- `wait=120`: checkpoint every 120 seconds instead of 60
- `log_size=0`: disable journal-size triggered checkpoints

Persistent configuration in `mongod.conf`:

```yaml
storage:
  wiredTiger:
    engineConfig:
      configString: "checkpoint=(wait=120)"
```

## Monitoring Checkpoint Duration

Long checkpoint durations are a sign of I/O bottlenecks:

```javascript
const txn = db.serverStatus().wiredTiger.transaction;
print("Last checkpoint (ms):", txn["transaction checkpoint most recent time (msecs)"]);
print("Max checkpoint (ms):",  txn["transaction checkpoint max time (msecs)"]);
```

Checkpoints regularly exceeding 1-2 seconds indicate storage I/O cannot keep up.

## Reducing Checkpoint I/O Impact

1. Use SSDs - NVMe drives reduce checkpoint latency dramatically
2. Separate journal and data on different volumes
3. Tune `dirty_target` to keep dirty page count low so each checkpoint has less to write:

```javascript
db.adminCommand({
  setParameter: 1,
  wiredTigerEngineRuntimeConfig:
    "eviction_dirty_target=2,eviction_dirty_trigger=10"
});
```

## Recovery Time Trade-off

Longer checkpoint intervals mean faster steady-state write performance but slower crash recovery - MongoDB must replay more journal entries. For most production workloads, the 60-second default is appropriate. Only increase it if you have strong SLAs on write latency and can tolerate slightly longer startup after a crash.

## Summary

WiredTiger checkpoints write in-memory state to disk on a timed or journal-size interval, providing crash recovery points. Frequent checkpoints reduce crash recovery time but add I/O pressure during normal operation. Tune checkpoint frequency, keep dirty page counts low through eviction thresholds, and use fast storage to minimize the latency impact. Monitor checkpoint duration metrics to detect storage I/O bottlenecks before they affect application write performance.
