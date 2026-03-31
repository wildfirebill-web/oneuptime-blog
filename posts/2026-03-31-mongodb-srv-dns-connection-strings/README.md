# How to Use SRV DNS Records with MongoDB Connection Strings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, SRV, DNS, Connection String, Atlas

Description: Learn how to use mongodb+srv:// connection strings to simplify cluster configuration through DNS SRV and TXT records for automatic discovery and TLS.

---

## What Are SRV Connection Strings?

The `mongodb+srv://` scheme uses DNS SRV and TXT records to automatically resolve MongoDB cluster members and connection options. Instead of hard-coding multiple hostnames and ports, a single DNS name resolves to all cluster members. This is the format used by MongoDB Atlas and simplifies connection string management.

## Standard vs SRV Connection String

```text
# Standard - manually list all hosts
mongodb://host1:27017,host2:27017,host3:27017/mydb?replicaSet=rs0&tls=true

# SRV - single hostname, everything resolved via DNS
mongodb+srv://user:pass@cluster.example.com/mydb
```

## How SRV Resolution Works

When the driver sees `mongodb+srv://`, it performs two DNS lookups:

```text
1. SRV record: _mongodb._tcp.cluster.example.com
   - Returns: host1:27017, host2:27017, host3:27017

2. TXT record: cluster.example.com
   - Returns: authSource=admin&replicaSet=rs0
```

The driver uses these records to build the equivalent standard connection string automatically.

## Checking SRV Records

```bash
# Check SRV records
dig +short SRV _mongodb._tcp.cluster.example.com

# Check TXT records for connection options
dig +short TXT cluster.example.com
```

## Using SRV in Node.js

```javascript
const { MongoClient } = require('mongodb');

// Single clean connection string - DNS handles the rest
const client = new MongoClient(
  'mongodb+srv://appuser:secret@cluster.mongodb.net/mydb',
  {
    retryWrites: true,
    w: 'majority',
  }
);

await client.connect();
const db = client.db('mydb');
```

## Using SRV in PyMongo

```python
from pymongo import MongoClient

client = MongoClient(
    "mongodb+srv://appuser:secret@cluster.mongodb.net/mydb",
    retryWrites=True,
    w="majority",
)
```

## Setting Up SRV Records (Self-Hosted)

For self-hosted deployments, add the following DNS records:

```text
; SRV records
_mongodb._tcp.mydb.example.com. 60 IN SRV 0 0 27017 mongo1.example.com.
_mongodb._tcp.mydb.example.com. 60 IN SRV 0 0 27017 mongo2.example.com.
_mongodb._tcp.mydb.example.com. 60 IN SRV 0 0 27017 mongo3.example.com.

; TXT record for connection options
mydb.example.com. 60 IN TXT "authSource=admin&replicaSet=myReplicaSet"
```

Then connect with:

```text
mongodb+srv://user:pass@mydb.example.com/myapp
```

## Limitations of SRV Strings

```text
- Cannot specify a port - SRV records define the port
- Cannot list multiple hosts - single hostname only
- Requires DNS infrastructure (not for localhost dev without hosts file tricks)
- DNS TTL affects how quickly cluster membership changes propagate
```

## SRV with Atlas

MongoDB Atlas generates an SRV connection string for every cluster. It is the recommended format and handles TLS, replica set name, and cluster member discovery automatically:

```text
mongodb+srv://user:pass@cluster0.abc123.mongodb.net/mydb?retryWrites=true&w=majority
```

## Summary

Use `mongodb+srv://` connection strings whenever possible. They simplify cluster management by centralizing connection configuration in DNS, support seamless cluster member changes without application redeployment, and automatically enable TLS. For MongoDB Atlas, the SRV connection string is the canonical format. For self-hosted deployments, SRV records are worth the DNS setup effort in any multi-node environment.
