# How to Set Up a 3-Node Replica Set in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, High Availability, Configuration, Deployment

Description: Step-by-step guide to setting up a 3-node MongoDB replica set for high availability, including configuration, initialization, and verification.

---

## Why a 3-Node Replica Set

A 3-node replica set is the minimum recommended configuration for production MongoDB deployments. It provides:
- One primary and two secondaries
- Automatic failover with a majority quorum (2 out of 3)
- Read scaling via secondary reads
- Zero data loss for acknowledged writes

## Prepare the Three Nodes

On each of the three servers, install MongoDB and create the data directory:

```bash
sudo apt-get install -y mongodb-org=7.0.0
sudo mkdir -p /data/db
sudo chown -R mongodb:mongodb /data/db
```

## Configure mongod.conf on Each Node

Edit `/etc/mongod.conf` on all three nodes. The replication section is the key change:

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0

storage:
  dbPath: /data/db

replication:
  replSetName: "rs0"

security:
  keyFile: /etc/mongodb/keyfile
```

Generate a shared keyfile for internal authentication between replica set members:

```bash
openssl rand -base64 756 > /etc/mongodb/keyfile
chmod 400 /etc/mongodb/keyfile
sudo chown mongodb:mongodb /etc/mongodb/keyfile
```

Copy the same keyfile to all three nodes.

## Start mongod on All Nodes

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

## Initialize the Replica Set

Connect to one node (the intended primary) and initialize:

```javascript
mongosh --host node1.example.com --port 27017

rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "node1.example.com:27017", priority: 2 },
    { _id: 1, host: "node2.example.com:27017", priority: 1 },
    { _id: 2, host: "node3.example.com:27017", priority: 1 }
  ]
})
```

Setting a higher priority on node1 ensures it becomes the primary after initialization.

## Create an Admin User

After the replica set initializes, create an admin user on the primary:

```javascript
use admin
db.createUser({
  user: "admin",
  pwd: "strongpassword",
  roles: [{ role: "root", db: "admin" }]
})
```

## Verify Replica Set Status

```javascript
rs.status()
```

Expected output shows one `PRIMARY` and two `SECONDARY` members:

```text
{
  "set": "rs0",
  "members": [
    { "name": "node1.example.com:27017", "stateStr": "PRIMARY" },
    { "name": "node2.example.com:27017", "stateStr": "SECONDARY" },
    { "name": "node3.example.com:27017", "stateStr": "SECONDARY" }
  ]
}
```

## Connect Using a Replica Set Connection String

```bash
mongosh "mongodb://admin:strongpassword@node1.example.com:27017,node2.example.com:27017,node3.example.com:27017/admin?replicaSet=rs0"
```

## Summary

A 3-node MongoDB replica set requires consistent `mongod.conf` with the same `replSetName` on all nodes, a shared keyfile for internal auth, and initialization via `rs.initiate()` with all three member hosts. The minimum 3-node configuration provides automatic failover and a write quorum. After initialization, verify with `rs.status()` to confirm one primary and two secondaries are healthy.
