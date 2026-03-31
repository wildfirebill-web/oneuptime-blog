# How to Use the connectTimeoutMS and socketTimeoutMS Options in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Timeout, Connection, Configuration, Driver

Description: Learn how to configure connectTimeoutMS and socketTimeoutMS in MongoDB to control connection establishment and socket-level timeouts for stable production apps.

---

## Understanding Timeout Options

MongoDB drivers expose several timeout parameters that control different phases of communication. Two of the most important are:

- `connectTimeoutMS`: how long the driver waits to establish a TCP connection to a MongoDB server
- `socketTimeoutMS`: how long the driver waits for a response on an already-established socket

Setting these correctly prevents applications from hanging indefinitely when a MongoDB server is unreachable or slow.

## Connection String Format

```text
mongodb://host:27017/mydb?connectTimeoutMS=10000&socketTimeoutMS=30000
```

## Node.js Driver

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient('mongodb://localhost:27017', {
  // Time to wait for TCP connection to be established (ms)
  connectTimeoutMS: 10000,

  // Time to wait for a response on an open socket (ms)
  socketTimeoutMS: 45000,

  // Time to wait for server selection from topology
  serverSelectionTimeoutMS: 5000,
});

await client.connect();
```

## PyMongo

```python
from pymongo import MongoClient

client = MongoClient(
    "mongodb://localhost:27017",
    connectTimeoutMS=10000,
    socketTimeoutMS=45000,
    serverSelectionTimeoutMS=5000,
)
```

## Java Driver

```java
import com.mongodb.MongoClientSettings;
import com.mongodb.connection.SocketSettings;
import java.util.concurrent.TimeUnit;

MongoClientSettings settings = MongoClientSettings.builder()
    .applyToSocketSettings(builder ->
        builder.connectTimeout(10, TimeUnit.SECONDS)
               .readTimeout(45, TimeUnit.SECONDS))
    .applyToClusterSettings(builder ->
        builder.serverSelectionTimeout(5, TimeUnit.SECONDS))
    .build();
```

## Difference Between socketTimeoutMS and serverSelectionTimeoutMS

```text
connectTimeoutMS:
  - Phase: TCP handshake
  - What it guards: initial connection establishment
  - Typical value: 5,000 - 30,000 ms

socketTimeoutMS:
  - Phase: waiting for data on an open connection
  - What it guards: long-running or stalled operations
  - Typical value: 30,000 - 120,000 ms (or 0 to disable)

serverSelectionTimeoutMS:
  - Phase: driver topology discovery
  - What it guards: finding a suitable server from the cluster
  - Typical value: 3,000 - 10,000 ms
```

## socketTimeoutMS = 0 (Disable Socket Timeout)

Setting `socketTimeoutMS=0` disables the socket timeout entirely, meaning the driver waits indefinitely. This is only appropriate for operations that are expected to run for a very long time, like large aggregations or bulk loads:

```javascript
const longRunningClient = new MongoClient(uri, {
  socketTimeoutMS: 0, // no timeout - for batch jobs only
});
```

For web applications, never use 0 - it can cause requests to hang forever.

## Detecting Timeout Errors

```javascript
const { MongoServerSelectionError, MongoNetworkError } = require('mongodb');

try {
  await client.connect();
  await db.collection('data').findOne({});
} catch (err) {
  if (err instanceof MongoServerSelectionError) {
    console.error('Could not select a server - check serverSelectionTimeoutMS');
  } else if (err instanceof MongoNetworkError) {
    console.error('Network timeout - check socketTimeoutMS and connectivity');
  }
}
```

## Recommended Production Values

```text
connectTimeoutMS:        10000   (10 seconds)
socketTimeoutMS:         45000   (45 seconds for typical web ops)
serverSelectionTimeoutMS: 5000   (5 seconds to find a server)
```

## Summary

Set `connectTimeoutMS` to a few seconds to fail fast when MongoDB is unreachable. Set `socketTimeoutMS` based on your slowest expected query - long enough to avoid false timeouts, short enough to surface genuinely hung operations. Always configure `serverSelectionTimeoutMS` to prevent applications from blocking during cluster topology changes.
