# How to Use the currentOp Command to Monitor Active Operations in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Operation, Monitoring, Performance, Diagnostic, Administration

Description: Learn how to use the currentOp command in MongoDB to monitor active database operations, identify slow queries, and diagnose lock contention.

---

## Introduction

The `currentOp` command returns a list of all operations currently executing on a MongoDB server. It is the primary tool for real-time visibility into what the database is doing. You can filter results to find slow queries, lock holders, and operations from specific clients or users.

## Basic Usage

Run `currentOp` to see all active operations:

```javascript
db.currentOp();
```

Or use the admin command form:

```javascript
db.adminCommand({ currentOp: 1 });
```

## Filtering Active Operations

Pass a filter document to narrow results:

```javascript
// Only active (not idle) operations
db.currentOp({ active: true });

// Operations running longer than 5 seconds
db.currentOp({ active: true, secs_running: { $gt: 5 } });

// Write operations only
db.currentOp({ active: true, op: { $in: ["update", "remove", "insert"] } });
```

## Inspecting a Specific Operation

An operation entry contains useful fields for diagnosis:

```javascript
{
  opid: 12345,
  op: "query",
  ns: "mydb.orders",
  secs_running: 45,
  client: "192.168.1.10:54321",
  desc: "conn123",
  waitingForLock: false,
  numYields: 200,
  locks: { Global: "r", Database: "r", Collection: "r" },
  command: { find: "orders", filter: { status: "pending" }, ... }
}
```

The `numYields` field shows how many times the operation yielded to allow other operations - very high values indicate a full collection scan.

## Finding Lock Waiters

Identify operations that are blocked waiting for a lock:

```javascript
db.currentOp({ waitingForLock: true });
```

## Monitoring a Specific Collection

```javascript
db.currentOp({
  active: true,
  "ns": "mydb.orders"
});
```

## Displaying a Real-Time Summary

Build a summary function for quick operator use:

```javascript
function opSummary() {
  const ops = db.currentOp({ active: true });
  const rows = ops.inprog.map(op => ({
    opid: op.opid,
    op: op.op,
    ns: op.ns,
    secs: op.secs_running || 0,
    client: op.client,
    waitingForLock: op.waitingForLock
  }));
  rows.sort((a, b) => b.secs - a.secs);
  rows.forEach(r => print(JSON.stringify(r)));
}

opSummary();
```

## Summary

The `currentOp` command provides real-time visibility into MongoDB's active operations. By filtering for long-running queries, lock waiters, and operations on specific namespaces, you can quickly diagnose performance bottlenecks and understand the root cause of slowdowns. Combining `currentOp` with `killOp` gives you full control over in-flight operations.
