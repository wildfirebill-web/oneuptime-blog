# How to Use the isMaster (hello) Command in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Command, Driver, Monitoring

Description: Learn how to use MongoDB's isMaster and hello commands to discover topology, check primary status, and retrieve connection metadata from any deployment.

---

## Overview

The `isMaster` command (renamed to `hello` in MongoDB 5.0) is a lightweight handshake command that MongoDB clients send to discover the role of the server they are connected to. It works on standalone nodes, replica set members, and mongos routers. You can run it manually in `mongosh` or use it in application code for health checks and topology inspection.

## Running the Command

In older MongoDB versions use `isMaster`:

```javascript
db.runCommand({ isMaster: 1 })
```

From MongoDB 5.0 onward, use `hello` (recommended):

```javascript
db.runCommand({ hello: 1 })
```

Both return the same shape of document. The `hello` command replaces the legacy name, which was considered non-inclusive language.

## Interpreting the Response

A typical response from a primary in a replica set:

```json
{
  "isWritablePrimary": true,
  "hosts": ["mongo1:27017", "mongo2:27017", "mongo3:27017"],
  "setName": "rs0",
  "setVersion": 1,
  "primary": "mongo1:27017",
  "me": "mongo1:27017",
  "electionId": "7fffffff0000000000000001",
  "lastWrite": {
    "opTime": { "ts": { "$timestamp": {} }, "t": 1 },
    "lastWriteDate": "2026-03-31T10:00:00.000Z"
  },
  "maxBsonObjectSize": 16777216,
  "maxMessageSizeBytes": 48000000,
  "maxWriteBatchSize": 100000,
  "localTime": "2026-03-31T10:00:01.000Z",
  "logicalSessionTimeoutMinutes": 30,
  "connectionId": 42,
  "minWireVersion": 0,
  "maxWireVersion": 21,
  "readOnly": false,
  "ok": 1
}
```

Key fields explained:

- `isWritablePrimary` - `true` when this node is the current primary
- `hosts` - all non-hidden members visible to clients
- `setName` - the replica set name
- `primary` - the current primary's hostname and port
- `maxBsonObjectSize` - maximum allowed document size (16 MB)
- `maxWireVersion` - protocol version used for feature negotiation by drivers

## Using hello for Health Checks

Because `hello` is a fast, unauthenticated command, it is ideal for liveness probes:

```bash
mongosh --quiet --eval 'db.runCommand({ hello: 1 }).isWritablePrimary' mongodb://mongo1:27017
```

A response of `true` confirms this node is the primary. A response of `false` means it is a secondary or arbiter.

## Checking From Application Code

In a Node.js application using the official driver:

```javascript
const { MongoClient } = require("mongodb");

async function checkPrimary(uri) {
  const client = new MongoClient(uri);
  await client.connect();
  const result = await client.db("admin").command({ hello: 1 });
  console.log("Is primary:", result.isWritablePrimary);
  console.log("Replica set:", result.setName);
  console.log("Wire version:", result.maxWireVersion);
  await client.close();
}

checkPrimary("mongodb://mongo1:27017").catch(console.error);
```

## Differences Between isMaster and hello

MongoDB 5.0 introduced `hello` as the canonical replacement:

```text
Legacy field        New field
-----------------------------------------
ismaster            isWritablePrimary
isMaster            hello
msg: "isdbgrid"     msg: "isdbgrid" (unchanged on mongos)
```

Drivers negotiating with MongoDB 5.0+ servers send `hello` automatically. If you maintain older monitoring scripts that check `ismaster: true`, update them to check `isWritablePrimary: true`.

## mongos Topology Discovery

On a mongos router, `hello` returns `msg: "isdbgrid"` to identify itself as part of a sharded cluster:

```javascript
db.runCommand({ hello: 1 }).msg; // "isdbgrid"
```

This lets drivers distinguish between a replica set and a sharded cluster without additional round trips.

## Summary

The `hello` command (formerly `isMaster`) is the fastest way to check the role of any MongoDB server and discover replica set topology. Use it for lightweight health checks, driver handshakes, and topology-aware scripting. Always prefer `hello` over `isMaster` when targeting MongoDB 5.0 or later.
