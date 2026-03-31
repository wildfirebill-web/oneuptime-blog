# How to Migrate from MongoDB 6.x to 7.x

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Upgrade, Migration, Version, Replica Set

Description: Step-by-step guide to upgrading MongoDB from version 6.x to 7.x, including new features, breaking changes, FCV management, and rolling upgrade procedures.

---

## Overview

MongoDB 7.0 is a Long Term Support (LTS) release introducing compound wildcard indexes, the auto-merge of chunks, performance improvements for time-series collections, and enhanced Atlas Search. This guide covers safely upgrading from 6.x to 7.0.

## Pre-Upgrade Checklist

```bash
# Confirm running latest 6.0 patch
mongosh --eval "db.version()"

# Confirm FCV
mongosh --eval "db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })"
```

You must be on MongoDB 6.0 before upgrading to 7.0. Skipping major versions is not supported.

## Driver Requirements for MongoDB 7.0

Minimum driver versions:

- Node.js driver: 5.0+
- PyMongo: 4.4+
- Java Sync driver: 4.10+
- Go driver: 1.12+

```bash
# Check and update your Node.js driver
npm list mongodb
npm install mongodb@6

# Update PyMongo
pip install --upgrade "pymongo>=4.4"
```

## Breaking Changes in MongoDB 7.0

MongoDB 7.0 removes some deprecated operators:

- `$where` is removed from the aggregation `$match` stage (use `$expr` instead)
- The legacy `mongo` shell is fully removed (use `mongosh`)

```javascript
// Remove this pattern
db.users.find({ $where: "this.age > 25" })

// Use $expr instead
db.users.find({ $expr: { $gt: ["$age", 25] } })
```

## Step 1: Set FCV to 6.0

```javascript
db.adminCommand({ setFeatureCompatibilityVersion: "6.0" })
```

Verify:

```javascript
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
// Expected: { version: "6.0" }
```

## Step 2: Upgrade Secondaries

```bash
# Stop each secondary, install 7.0, and restart
sudo systemctl stop mongod

# Ubuntu/Debian
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt-get update && sudo apt-get install -y mongodb-org

sudo systemctl start mongod
```

Confirm secondary is healthy before proceeding:

```javascript
rs.status()  // Verify member state is SECONDARY
```

## Step 3: Step Down and Upgrade the Primary

```javascript
rs.stepDown()
```

Upgrade the original primary the same way as the secondaries.

## Step 4: Set FCV to 7.0

```javascript
db.adminCommand({ setFeatureCompatibilityVersion: "7.0" })
```

## Step 5: Verify and Use New Features

```javascript
db.version()  // "7.0.x"

// Use compound wildcard indexes (new in 7.0)
db.products.createIndex({ "$**": 1, category: 1 })

// Use time series improvements
db.createCollection("sensor_data", {
  timeseries: {
    timeField: "ts",
    metaField: "sensor",
    granularity: "seconds"
  }
})
```

## Post-Upgrade Monitoring

After upgrading, monitor for any performance regressions:

```javascript
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find().sort({ ts: -1 }).limit(10)
```

## Summary

MongoDB 7.0 is an LTS release with meaningful performance improvements and new index types. Upgrade by setting FCV to 6.0, performing a rolling upgrade of secondaries then the primary, and advancing FCV to 7.0. Remove any usage of the legacy `$where` operator and ensure all clients are on driver versions supporting 7.0 before upgrading.
