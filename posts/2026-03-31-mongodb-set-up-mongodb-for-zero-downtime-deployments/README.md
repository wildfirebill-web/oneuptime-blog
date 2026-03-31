# How to Set Up MongoDB for Zero-Downtime Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Deployment, Replica Set, High Availability, Operation

Description: Learn how to configure MongoDB replica sets, rolling upgrades, and application-level strategies to achieve zero-downtime deployments in production.

---

Zero-downtime deployments with MongoDB require careful coordination between your database configuration, replica set setup, and application layer. The key is ensuring that at least one primary node is always available to serve writes while maintenance or upgrades occur on other members.

## Configure a Replica Set for High Availability

A replica set with at least three members is the foundation of zero-downtime MongoDB deployments. This ensures that a new primary can be elected even if one member goes offline.

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017", priority: 2 },
    { _id: 1, host: "mongo2:27017", priority: 1 },
    { _id: 2, host: "mongo3:27017", priority: 1 }
  ]
})
```

Set higher priority on your preferred primary to reduce unintended failovers during routine operations.

## Use Rolling Restarts for Upgrades

When upgrading MongoDB or changing configurations, always perform rolling restarts - one node at a time - starting with secondaries.

```bash
# Step 1: Restart a secondary
mongosh --host mongo2:27017 --eval "db.adminCommand({ shutdown: 1 })"

# Step 2: Wait for it to rejoin the replica set, then check status
mongosh --host mongo1:27017 --eval "rs.status()"

# Step 3: Step down the primary gracefully before restarting it
mongosh --host mongo1:27017 --eval "rs.stepDown(60)"
```

Always wait for the restarted secondary to fully sync before proceeding to the next node.

## Configure Write Concern for Safe Operations

Use majority write concern so that write operations are acknowledged only after they have been replicated to the majority of voting members.

```javascript
const client = new MongoClient(uri, {
  writeConcern: {
    w: "majority",
    wtimeoutMS: 5000
  }
});
```

This ensures no data is lost if the primary fails immediately after an acknowledged write.

## Enable Retryable Writes in Your Driver

Retryable writes allow MongoDB drivers to automatically retry certain write operations after a transient network error or primary failover.

```javascript
const client = new MongoClient(
  "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0",
  {
    retryWrites: true,
    retryReads: true
  }
);
```

This is enabled by default in modern MongoDB drivers but worth verifying in older driver versions.

## Monitor Replication Lag Before and After Deployments

Before taking down a secondary for maintenance, confirm that it is fully caught up with the primary.

```javascript
// Check replication lag on each secondary
const status = rs.status();
status.members.forEach(member => {
  if (member.stateStr === "SECONDARY") {
    const lagSeconds = (new Date() - member.optimeDate) / 1000;
    print(`${member.name} lag: ${lagSeconds}s`);
  }
});
```

A lag greater than a few seconds indicates the secondary has not finished syncing and should not be taken offline yet.

## Use Connection String Options for Resilience

Configure your connection string with appropriate timeout and server selection settings so your application handles primary transitions gracefully.

```text
mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0&serverSelectionTimeoutMS=10000&socketTimeoutMS=30000&connectTimeoutMS=10000
```

Combining these options with retryable writes and majority write concern gives your application the best chance of handling a failover without visible errors.

## Summary

Zero-downtime MongoDB deployments depend on a properly configured replica set with at least three members, rolling restart procedures that touch secondaries before the primary, majority write concern, and retryable writes enabled in the driver. Monitoring replication lag before each maintenance step and using resilient connection string settings ensures your application remains available throughout deployment operations.
