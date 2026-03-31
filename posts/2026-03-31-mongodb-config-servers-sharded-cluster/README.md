# How to Configure Config Servers for a MongoDB Sharded Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Config Server, Replica Set, Database

Description: Learn how to deploy and configure config servers for a MongoDB sharded cluster, including replica set setup, connection options, and production best practices.

---

Config servers store the metadata for a sharded cluster - chunk ranges, shard locations, and cluster configuration. They must run as a replica set (CSRS) with at least three members for production deployments.

## Deploy a Config Server Replica Set

Each config server node requires `configsvr: true` in its configuration file.

```yaml
# /etc/mongod-cfg1.conf
sharding:
  clusterRole: configsvr
replication:
  replSetName: configReplSet
net:
  port: 27019
  bindIp: 0.0.0.0
storage:
  dbPath: /var/lib/mongodb-config
```

Start the config server process on each node:

```bash
mongod --config /etc/mongod-cfg1.conf
```

Repeat on all three config server nodes (ports 27019, 27019, 27019 across different hosts).

## Initialize the Config Server Replica Set

Connect to one config server and initiate the replica set:

```javascript
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "cfg1.example.com:27019" },
    { _id: 1, host: "cfg2.example.com:27019" },
    { _id: 2, host: "cfg3.example.com:27019" }
  ]
})
```

Verify the replica set is healthy:

```javascript
rs.status()
```

Wait until one member reports `"stateStr": "PRIMARY"` and the others show `"stateStr": "SECONDARY"`.

## Connect mongos to the Config Servers

Once the config replica set is up, launch `mongos` and point it to the config servers:

```yaml
# /etc/mongos.conf
sharding:
  configDB: configReplSet/cfg1.example.com:27019,cfg2.example.com:27019,cfg3.example.com:27019
net:
  port: 27017
  bindIp: 0.0.0.0
```

```bash
mongos --config /etc/mongos.conf
```

## Verify Config Server Metadata

Connect to `mongos` and inspect the cluster configuration:

```javascript
use config
db.shards.find().pretty()
db.databases.find().pretty()
db.chunks.find().limit(5).pretty()
```

## Production Considerations

Keep config servers on dedicated hardware or VMs separate from shard data nodes. Config server data is small but critical - losing a majority of config servers makes the cluster read-only and prevents chunk migrations.

Enable journaling and use a fast storage layer for config servers:

```yaml
storage:
  dbPath: /var/lib/mongodb-config
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
```

Back up config server data regularly using `mongodump` against the config replica set:

```bash
mongodump --host configReplSet/cfg1.example.com:27019 \
  --db config \
  --out /backup/config-$(date +%F)
```

## Monitoring Config Server Health

Watch for replication lag and connectivity from `mongos`:

```javascript
// From mongos shell
db.adminCommand({ getShardMap: 1 })
```

Use `rs.printReplicationInfo()` on the config replica set to monitor oplog status.

## Summary

Config servers are the backbone of a MongoDB sharded cluster, storing all metadata about chunk distribution and shard membership. Deploying them as a three-member replica set ensures high availability and prevents data loss. Keep them monitored, backed up, and isolated from shard workloads.
