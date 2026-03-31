# How to Migrate from MongoDB 4.x to 5.x

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Upgrade, Migration, Version, Replica Set

Description: Step-by-step guide to upgrading a MongoDB replica set from version 4.x to 5.x, including compatibility checks, feature compatibility version changes, and rollback steps.

---

## Overview

MongoDB 5.0 introduced time series collections, live resharding, and serverless Atlas instances. Before upgrading from 4.x, you must set the Feature Compatibility Version (FCV) to 4.4 and ensure all drivers and tools support MongoDB 5.0.

## Pre-Upgrade Checklist

Before starting:

```bash
# Check current MongoDB version
mongosh --eval "db.version()"

# Check current feature compatibility version
mongosh --eval "db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })"

# Check for deprecated operators or configurations
mongosh --eval "db.adminCommand({ getLog: 'global' })" | grep -i "deprecated"
```

Ensure you are running the latest MongoDB 4.4 patch release before upgrading to 5.0.

## Step 1: Update Driver and Application Compatibility

Verify your MongoDB drivers support version 5.0:

- Node.js driver: 4.0+
- PyMongo: 3.12+
- Java driver: 4.3+
- Go driver: 1.7+

Update driver versions in your application before proceeding.

## Step 2: Set Feature Compatibility Version to 4.4

```javascript
db.adminCommand({ setFeatureCompatibilityVersion: "4.4" })
```

Verify:

```javascript
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
// Expected: { featureCompatibilityVersion: { version: "4.4" } }
```

## Step 3: Upgrade Secondaries First

For a replica set, upgrade each secondary before the primary:

```bash
# On each secondary - stop mongod
sudo systemctl stop mongod

# Install MongoDB 5.0
wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org=5.0.0

# Start mongod
sudo systemctl start mongod
```

Wait for the secondary to reach `SECONDARY` state:

```javascript
rs.status()
```

## Step 4: Step Down and Upgrade the Primary

```javascript
// Force an election to step down the primary
rs.stepDown()
```

Then upgrade the original primary using the same steps as the secondaries.

## Step 5: Set Feature Compatibility Version to 5.0

After all members are on 5.0, upgrade the FCV:

```javascript
db.adminCommand({ setFeatureCompatibilityVersion: "5.0" })
```

## Step 6: Verify the Upgrade

```javascript
db.version()                    // Should return "5.0.x"
rs.status()                     // All members should be healthy
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
// Expected: { version: "5.0" }
```

## Rollback Plan

If issues arise, you can downgrade back to 4.4 only if FCV has not been set to 5.0:

```javascript
// Only possible if still at FCV 4.4
db.adminCommand({ setFeatureCompatibilityVersion: "4.4" })
```

Then reinstall MongoDB 4.4 on each member.

## Summary

Upgrading MongoDB from 4.x to 5.x requires setting the FCV to 4.4 first, upgrading secondaries before the primary, and then advancing the FCV to 5.0 after all members are updated. Update your drivers before upgrading the server, and always test the upgrade on a non-production replica set first.
