# How to Configure an Arbiter in a MongoDB Replica Set

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Arbiter, Election, Operation

Description: Add a MongoDB arbiter to a two-member replica set to enable elections without storing a full data copy on the third node.

---

## What Is an Arbiter

An arbiter is a replica set member that participates in elections but does not hold a copy of the data. It exists solely to provide a tiebreaker vote when a replica set has an even number of data-bearing members, preventing split-brain situations where no primary can be elected.

Use an arbiter when you want fault tolerance but cannot afford a full third data node - for example, a two-server setup where a third full copy is cost-prohibitive.

## When to Use (and Avoid) an Arbiter

Use an arbiter for a 2-data-node + 1-arbiter configuration where storing a third full data copy is too expensive.

Avoid arbiters for replica sets that will store large datasets, use transactions at scale, or need strong data durability guarantees. A three-node data replica set is always preferred.

## Start the Arbiter mongod

The arbiter needs its own data directory (even though it stores no user data) and the replica set name.

```bash
mongod --replSet myReplicaSet \
  --dbpath /data/arbiter \
  --port 27023 \
  --bind_ip localhost,192.168.1.16 \
  --logpath /var/log/mongodb/arbiter.log \
  --fork
```

The arbiter's data directory will remain very small - only replica set metadata is stored.

## Add the Arbiter to the Replica Set

Connect to the primary and add the arbiter using `rs.addArb()`:

```javascript
mongosh --host 192.168.1.10 --port 27017
```

```javascript
rs.addArb("192.168.1.16:27023");
```

Alternatively use the full form:

```javascript
rs.add({
  host: "192.168.1.16:27023",
  arbiterOnly: true
});
```

## Verify the Arbiter

Check that the arbiter appears correctly in the replica set configuration:

```javascript
rs.conf().members
```

The arbiter member should show `arbiterOnly: true`. In `rs.status()`, its `stateStr` should be `"ARBITER"`.

```javascript
rs.status().members.filter(m => m.stateStr === "ARBITER")
```

## Arbiter Security Considerations

Arbiters should still be secured even though they hold no user data. They participate in network communication with data nodes, so enable TLS and authentication:

```yaml
# /etc/mongod-arbiter.conf
security:
  keyFile: /etc/mongodb/keyfile
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb-arbiter.pem
    CAFile: /etc/ssl/mongodbCA.crt
replication:
  replSetName: "myReplicaSet"
```

## Summary

An arbiter lets you run a MongoDB replica set with two data-bearing nodes plus one lightweight voting node, enabling elections without the cost of a third full data copy. Add one with `rs.addArb()`. While useful for cost-constrained deployments, prefer full three-member data replica sets for production workloads requiring strong durability or transaction support.
