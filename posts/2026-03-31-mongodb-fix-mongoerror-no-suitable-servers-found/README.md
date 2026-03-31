# How to Fix MongoError: No Suitable Servers Found in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Server Selection, Replica Set, Error, Troubleshooting

Description: Learn why MongoError No Suitable Servers Found occurs during server selection and how to fix it by tuning timeouts, replica set config, and read preferences.

---

## Understanding the Error

`MongoServerSelectionError: No suitable servers found (serverSelectionTimeoutMS: 30000)` means the driver could not find any MongoDB server that matches the requested criteria within the server selection timeout. The driver has a topology description but none of the servers match the required state.

```text
MongoServerSelectionError: No suitable servers found (`serverSelectionTimeoutMS` expired)
    [Server{ address: localhost:27017, type: Unknown, state: <no server> }]
```

## Common Causes and Fixes

### 1. MongoDB Is Not Running

The simplest cause - verify the server is running:

```bash
sudo systemctl status mongod
sudo systemctl start mongod
```

### 2. Replica Set Not Fully Initialized

If you configured a replica set URI but the replica set has not been initialized, no member has a defined role yet:

```javascript
// Initialize the replica set
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})
```

Check status:

```javascript
rs.status()
```

### 3. Read Preference Mismatch

Using `readPreference: 'secondary'` when no secondaries are available causes "no suitable servers":

```javascript
// Falls back gracefully if no secondary is available
const client = new MongoClient(uri, {
  readPreference: 'secondaryPreferred' // uses primary if no secondary
});
```

Or query with a specific preference:

```javascript
await db.collection('reports')
  .find({})
  .withReadPreference(ReadPreference.SECONDARY_PREFERRED)
  .toArray();
```

### 4. Wrong Replica Set Name

The `replicaSet` parameter in the URI must match the actual `_id` configured in `rs.conf()`:

```javascript
// Check replica set name
rs.conf()._id // should match what's in your URI

// URI must match exactly
const uri = 'mongodb://host1:27017,host2:27017/?replicaSet=rs0';
```

### 5. Network Partitions or Firewall Blocking Discovery

The driver uses a heartbeat to monitor server state. If the heartbeat fails due to firewall rules or network issues:

```bash
# Test connectivity to each replica set member
nc -zv mongo1 27017
nc -zv mongo2 27017
nc -zv mongo3 27017
```

### 6. serverSelectionTimeoutMS Too Low

In high-latency environments, the default 30 seconds may not be enough during startup:

```javascript
const client = new MongoClient(uri, {
  serverSelectionTimeoutMS: 60000, // 60 seconds
  connectTimeoutMS: 10000,
  socketTimeoutMS: 45000
});
```

### 7. Atlas IP Access List

For Atlas deployments, ensure your application's IP is in the Network Access allow list. Atlas will refuse connections from unlisted IPs, causing server selection to time out.

```bash
# Test Atlas connectivity
mongosh "mongodb+srv://cluster0.example.mongodb.net/test" --username myuser
```

## Debugging with Monitoring Events

```javascript
client.on('serverDescriptionChanged', (event) => {
  console.log('Server description changed:', JSON.stringify(event));
});
client.on('topologyDescriptionChanged', (event) => {
  console.log('Topology changed:', JSON.stringify(event));
});
```

## Summary

`No Suitable Servers Found` occurs when the driver cannot find a server matching the required role (primary, secondary, etc.) within the timeout. Check that MongoDB is running, the replica set is initialized with the correct name, read preferences are achievable, firewall rules allow the heartbeat port, and Atlas IP allowlists are configured correctly.
