# How to Troubleshoot MongoDB Connection Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Connection, Troubleshooting, Network, Driver

Description: Diagnose and fix MongoDB connection failures by checking network access, authentication, TLS settings, connection pool exhaustion, and driver configuration.

---

## Overview

MongoDB connection issues surface as timeouts, authentication errors, refused connections, or pool exhaustion. Each has a distinct cause and fix. A systematic diagnostic approach covers network reachability, authentication, TLS, and application-level pool settings.

## Step 1: Verify Network Reachability

```bash
# Test basic TCP connectivity
nc -zv <hostname> 27017

# Or using telnet
telnet <hostname> 27017

# If using Atlas, verify your IP is in the access list
atlas accessLists list --projectId <PROJECT_ID>
```

## Step 2: Test with mongosh

Isolate whether the issue is the driver or the server.

```bash
# Test connection with full URI
mongosh "mongodb+srv://user:password@cluster.mongodb.net/mydb" \
  --eval 'db.runCommand({ ping: 1 })'

# Test without TLS (self-hosted)
mongosh --host localhost --port 27017 \
  --username admin --password secret \
  --authenticationDatabase admin \
  --eval 'db.adminCommand({ ping: 1 })'
```

## Step 3: Common Error Messages and Fixes

**"Connection refused" (ECONNREFUSED)**

```bash
# MongoDB is not running or is on a different port
systemctl status mongod
ss -tlnp | grep 27017

# Check bindIp in mongod.conf - must include the client's address
grep bindIp /etc/mongod.conf
# Change to 0.0.0.0 to accept all interfaces (restrict with firewall rules)
```

**"Authentication failed"**

```javascript
// Verify user exists and roles are correct
use admin
db.getUser('myuser')

// Reset password if needed
db.changeUserPassword('myuser', 'newpassword')
```

**"SSL/TLS handshake failed"**

```bash
# Test TLS with openssl
openssl s_client -connect <hostname>:27017 -tls1_2

# Check certificate expiry
openssl s_client -connect <hostname>:27017 2>/dev/null | \
  openssl x509 -noout -dates
```

**"Timed out waiting for a usable server"**

This indicates the driver cannot reach any replica set member. Check that all members are reachable and that the replica set name in the URI matches.

```javascript
// In Node.js driver - print server topology errors
const client = new MongoClient(uri, {
  serverSelectionTimeoutMS: 5000
});

try {
  await client.connect();
} catch (err) {
  console.error('Server selection failed:', err.reason?.servers);
}
```

## Step 4: Diagnose Connection Pool Exhaustion

Pool exhaustion occurs when all connections are in use and new requests wait until timeout.

```javascript
// Check current connections on the server
db.runCommand({ serverStatus: 1 }).connections

// In the application driver, add event listeners to monitor the pool
client.on('connectionPoolCreated', (event) => console.log('Pool created'));
client.on('connectionCheckedOut', (event) => console.log('Connection checked out'));
client.on('connectionCheckOutFailed', (event) => console.error('Pool exhausted:', event.reason));
```

Increase the pool size or reduce query execution time to release connections faster.

```javascript
const client = new MongoClient(uri, {
  maxPoolSize: 50,
  waitQueueTimeoutMS: 5000,   // How long to wait for a pool connection
  socketTimeoutMS: 30000,     // How long to wait for a query response
  connectTimeoutMS: 10000     // How long to wait for initial connection
});
```

## Step 5: Atlas-Specific Connection Diagnostics

```bash
# Verify Atlas cluster status
atlas clusters describe my-cluster --projectId <PROJECT_ID>

# Check IP access list
atlas accessLists list --projectId <PROJECT_ID>

# Add your current IP to the access list
atlas accessLists create \
  --currentIp \
  --projectId <PROJECT_ID>

# Verify database user exists
atlas dbusers list --projectId <PROJECT_ID>
```

## Step 6: Enable Driver Debug Logging

```javascript
// Enable MongoDB driver debug logging in Node.js
const { MongoClient } = require('mongodb');
const client = new MongoClient(uri, {
  monitorCommands: true
});

client.on('commandStarted', (event) => {
  console.debug('Command started:', event.commandName, event.connectionId);
});

client.on('commandFailed', (event) => {
  console.error('Command failed:', event.commandName, event.failure);
});
```

## Summary

MongoDB connection issues fall into four categories: network reachability, authentication failures, TLS errors, and connection pool exhaustion. Start with `nc` or `mongosh` to isolate network and credential problems, check Atlas IP access lists for Atlas deployments, tune `maxPoolSize` and timeout settings in the driver for pool exhaustion, and use driver event listeners to observe real-time connection pool behavior.
