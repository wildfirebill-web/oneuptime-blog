# How to Use mongosh for Database Administration Tasks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongosh, Administration, Monitoring

Description: Learn how to perform common MongoDB DBA tasks in mongosh - check server status, manage indexes, kill operations, rotate logs, and monitor replica sets.

---

## Server Status and Diagnostics

```javascript
// Overall server status
db.adminCommand("serverStatus");

// Key metrics snapshot
const status = db.adminCommand("serverStatus");
printjson({
  connections: status.connections,
  memory:      status.mem,
  opcounters:  status.opcounters
});

// Current MongoDB version
db.version();
db.adminCommand("buildInfo").version;
```

## Database and Collection Management

```javascript
// List all databases
show dbs;

// Database statistics
db.stats();

// List collections in current database
show collections;
db.getCollectionNames();

// Collection statistics
db.orders.stats();

// Drop a collection
db.tempData.drop();

// Drop a database
use olddb
db.dropDatabase();
```

## Index Management

```javascript
// List indexes on a collection
db.orders.getIndexes();

// Create an index
db.orders.createIndex({ status: 1, createdAt: -1 });

// Drop an index by name
db.orders.dropIndex("status_1_createdAt_-1");

// Check index build progress
db.adminCommand({ currentOp: true, $or: [{ "command.createIndexes": { $exists: true } }] });
```

## Monitoring Active Operations

```javascript
// Show all current operations
db.adminCommand({ currentOp: true });

// Find slow operations (> 5 seconds)
db.adminCommand({
  currentOp: true,
  secs_running: { $gt: 5 }
});
```

## Killing a Long-Running Operation

```javascript
// Get the operation ID from currentOp output
const ops = db.adminCommand({ currentOp: true, secs_running: { $gt: 10 } });
const opId = ops.inprog[0].opid;

// Kill the operation
db.adminCommand({ killOp: 1, op: opId });
```

## Log Management

```javascript
// Rotate log files
db.adminCommand({ logRotate: 1 });

// Get recent log entries
db.adminCommand({ getLog: "global" }).log.slice(-20).forEach(print);

// Change log verbosity
db.adminCommand({ setParameter: 1, logLevel: 1 });
```

## Replica Set Administration

```javascript
// Check replica set status
rs.status();

// Check replication lag
const status = rs.status();
status.members.forEach(m => {
  print(`${m.name}: ${m.stateStr}, lag: ${m.optimeDate}`);
});

// Step down primary (triggers election)
rs.stepDown(60); // 60-second step-down period
```

## Profiler Management

```javascript
// Enable slow query profiling (threshold: 100ms)
db.setProfilingLevel(1, { slowms: 100 });

// Check profiler settings
db.getProfilingStatus();

// Query profiler data
db.system.profile.find({ millis: { $gt: 200 } }).sort({ ts: -1 }).limit(5);
```

## Running Admin Commands

```javascript
// Flush writes to disk
db.adminCommand({ fsync: 1 });

// Compact a collection (reduces fragmentation)
db.runCommand({ compact: "orders" });

// Validate a collection
db.runCommand({ validate: "orders", full: true });
```

## Summary

mongosh gives DBAs direct access to all MongoDB administrative operations - from server status checks and index management to killing runaway queries and rotating logs. Use `db.adminCommand()` for low-level control, `rs.status()` for replication monitoring, and `db.setProfilingLevel()` to enable slow query analysis. Combine these tools to diagnose and resolve production issues quickly.
