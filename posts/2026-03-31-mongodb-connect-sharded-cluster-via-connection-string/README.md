# How to Connect to a MongoDB Sharded Cluster via Connection String

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Connection, Cluster, Driver

Description: Learn how to connect to a MongoDB sharded cluster via connection string by targeting mongos routers with proper authentication and timeout configuration.

---

Applications connect to a sharded MongoDB cluster through `mongos` query routers, not directly to individual shards. The connection string points to one or more `mongos` instances, and the routers handle routing queries to the correct shards.

## Basic Sharded Cluster Connection String

```text
mongodb://mongos1:27017,mongos2:27017/myDatabase
```

Always include at least two `mongos` instances for high availability. If one router becomes unavailable, the driver automatically uses the other.

## With Authentication

```text
mongodb://appUser:password@mongos1:27017,mongos2:27017/myDatabase?authSource=admin
```

User accounts on a sharded cluster are stored in the `admin` database on the config servers and propagated to all shards.

## Full Production Connection String

```text
mongodb://appUser:password@mongos1:27017,mongos2:27017/myDatabase?authSource=admin&tls=true&tlsCAFile=/etc/ssl/ca.pem&w=majority&retryWrites=true&maxPoolSize=50&serverSelectionTimeoutMS=15000
```

## Node.js Connection

```javascript
const { MongoClient } = require("mongodb");

const uri =
  "mongodb://appUser:password@mongos1:27017,mongos2:27017/myDB" +
  "?authSource=admin&retryWrites=true&w=majority";

const client = new MongoClient(uri, {
  maxPoolSize: 50,
  serverSelectionTimeoutMS: 15000,
  connectTimeoutMS: 10000
});

await client.connect();

// Verify cluster topology
const adminDb = client.db("admin");
const status = await adminDb.command({ serverStatus: 1 });
console.log("Process:", status.process); // Should be "mongos"
```

## Python with PyMongo

```python
from pymongo import MongoClient

client = MongoClient(
    "mongodb://appUser:password@mongos1:27017,mongos2:27017/"
    "?authSource=admin&retryWrites=true&w=majority",
    maxPoolSize=50,
    serverSelectionTimeoutMS=15000
)

# Confirm connection to mongos
server_info = client.server_info()
print(f"Connected to MongoDB {server_info['version']}")
```

## Difference from Replica Set Connection

A sharded cluster connection does NOT use the `replicaSet` parameter:

```text
# Replica set (includes replicaSet=)
mongodb://host1:27017,host2:27017,host3:27017/myDB?replicaSet=rs0

# Sharded cluster (no replicaSet= - connects to mongos)
mongodb://mongos1:27017,mongos2:27017/myDB
```

If you accidentally include `replicaSet=` when connecting to `mongos`, the driver will fail to establish the connection.

## Read Preferences on Sharded Clusters

Read preferences work differently on sharded clusters:

```text
mongodb://mongos1:27017,mongos2:27017/myDB?readPreference=secondaryPreferred
```

The `mongos` router forwards the read preference to the appropriate shard replica set member. However, be aware that non-primary reads on sharded clusters can return stale data and may miss documents in mid-migration chunks.

## Targeting a Specific Mongos in Code

For debugging, connect to a single `mongos`:

```javascript
// Direct mongos connection for admin tasks
const adminClient = new MongoClient("mongodb://mongos1:27017");
await adminClient.connect();
const result = await adminClient.db("admin").command({ listShards: 1 });
console.log("Shards:", result.shards);
```

## Checking Cluster Status

```javascript
// Run against a mongos connection
db.adminCommand({ serverStatus: 1 })
// process: "mongos" confirms you are connected to a router

sh.status()    // Shard distribution, chunk counts, balancer state
sh.isBalancerRunning()
```

## Connection Pool Sizing for Sharded Clusters

Each `mongos` maintains its own connection pool to each shard. To avoid overwhelming shards:

```text
mongos_count * maxPoolSize <= shard_max_incoming_connections / 2
```

For example, with 3 `mongos` instances and 200 max shard connections:

```text
3 * maxPoolSize <= 200 / 2 = 100
maxPoolSize <= 33
```

## Summary

Connect to a sharded cluster by pointing the URI at `mongos` routers without the `replicaSet` parameter. Include at least two `mongos` hosts for resilience. Size connection pools to avoid overwhelming shard members, and verify you are connected to `mongos` (not a shard) using `db.adminCommand({ serverStatus: 1 })`.
