# How to Migrate from MongoDB 5.x to 6.x

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Upgrade, Migration, Version, Replica Set

Description: Step-by-step guide to upgrading a MongoDB replica set from version 5.x to 6.x, covering new aggregation operators, driver updates, and the rolling upgrade process.

---

## Overview

MongoDB 6.0 introduced new aggregation operators (`$densify`, `$fill`, `$setWindowFields` enhancements), encrypted fields with Queryable Encryption, and cluster-to-cluster sync. This guide walks through safely upgrading from 5.x to 6.0.

## Pre-Upgrade Checklist

```bash
# Confirm current version
mongosh --eval "db.version()"

# Confirm FCV is at 5.0 (or set it now)
mongosh --eval "db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })"
```

Ensure you are on the latest MongoDB 5.0 patch release before upgrading.

## Step 1: Update Drivers for MongoDB 6.0 Compatibility

Minimum driver versions for MongoDB 6.0:

- Node.js driver: 4.9+
- PyMongo: 4.1+
- Java driver: 4.7+
- Go driver: 1.10+

```bash
# Update Node.js driver
npm install mongodb@latest

# Update PyMongo
pip install --upgrade pymongo
```

## Step 2: Check for Removed Features

MongoDB 6.0 removed `map-reduce` as a standalone command (use aggregation with `$group` instead) and deprecated `count()` in favor of `countDocuments()` and `estimatedDocumentCount()`:

```javascript
// Deprecated - do not use in 6.x
db.orders.count({ status: "completed" })

// Use this instead
db.orders.countDocuments({ status: "completed" })
db.orders.estimatedDocumentCount()
```

## Step 3: Set FCV to 5.0

```javascript
db.adminCommand({ setFeatureCompatibilityVersion: "5.0" })
```

## Step 4: Rolling Upgrade - Secondaries First

```bash
# Stop mongod on each secondary
sudo systemctl stop mongod

# Install MongoDB 6.0 on Ubuntu
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt-get update && sudo apt-get install -y mongodb-org

# Restart
sudo systemctl start mongod
```

Wait for each secondary to return to `SECONDARY` state before proceeding to the next.

## Step 5: Upgrade the Primary

```javascript
rs.stepDown()  // Trigger election
```

Then upgrade the original primary the same way as the secondaries.

## Step 6: Enable FCV 6.0

```javascript
db.adminCommand({ setFeatureCompatibilityVersion: "6.0" })
```

## Step 7: Verify

```javascript
db.version()   // "6.0.x"
rs.status()    // All members healthy
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

## Using New 6.0 Aggregation Features

After upgrading, you can leverage new operators:

```javascript
// $fill - fill in missing values in a time series
db.sensor_data.aggregate([
  { $fill: {
      sortBy: { ts: 1 },
      output: { temperature: { method: "linear" } }
  }}
])
```

## Summary

Upgrading MongoDB from 5.x to 6.x follows the same rolling upgrade pattern: update drivers, set FCV, upgrade secondaries one by one, step down the primary, upgrade it, then advance FCV. MongoDB 6.0 removes `map-reduce` and deprecates `count()` - audit your application code for these before upgrading.
