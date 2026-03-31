# How to Check MongoDB Version and Feature Compatibility

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Version Management, Administration

Description: Learn how to check the MongoDB server version and featureCompatibilityVersion (FCV) using mongosh to prepare for upgrades and verify compatibility.

---

Before upgrading or downgrading MongoDB, you need to know the current binary version and the `featureCompatibilityVersion` (FCV). These two values together determine what features your cluster can use and what upgrade paths are available.

## Checking the MongoDB Binary Version

Connect with `mongosh` and run:

```javascript
db.version()
```

Output:

```text
7.0.5
```

Or use `serverStatus`:

```javascript
db.adminCommand({ serverStatus: 1 }).version
```

From the shell:

```bash
mongod --version
```

Output:

```text
db version v7.0.5
Build Info: {
  "version": "7.0.5",
  "gitVersion": "abc123",
  "openSSLVersion": "OpenSSL 3.0.2",
  ...
}
```

## Checking featureCompatibilityVersion

The FCV controls which version-specific features are enabled, independently of the binary version:

```javascript
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

Output:

```text
{
  featureCompatibilityVersion: { version: '7.0' },
  ok: 1
}
```

## Understanding FCV vs Binary Version

| Scenario | Binary Version | FCV |
|----------|---------------|-----|
| Freshly installed 7.0 | 7.0 | 7.0 |
| Upgraded binary, FCV not yet bumped | 7.0 | 6.0 |
| During downgrade preparation | 6.0 | 6.0 |

Running a 7.0 binary with FCV 6.0 lets you use the new binary but keeps the on-disk format compatible with 6.0, allowing a downgrade if needed.

## Checking FCV on All Replica Set Members

From the primary, check FCV on each member:

```javascript
rs.status().members.forEach(function(m) {
  print(m.name, m.stateStr)
})
```

Then connect to each secondary and check FCV individually:

```bash
mongosh --host secondary-host:27017 --eval "db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })"
```

## Checking FCV in a Sharded Cluster

On a sharded cluster, check FCV on each component:

```bash
# Check mongos
mongosh --host mongos-host:27017 --eval "db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })"

# Check each shard primary
mongosh --host shard1-host:27017 --eval "db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })"
```

All components must have the same FCV before upgrading.

## Version Compatibility Matrix

MongoDB supports upgrading one major version at a time:

```text
5.0 -> 6.0 -> 7.0  (correct)
5.0 -> 7.0          (not supported, skip upgrade)
```

Always upgrade incrementally through each major version.

## Checking Driver Compatibility

Check which server versions your driver supports:

```bash
# For Node.js driver
npm info mongodb versions | tr ',' '\n' | tail -5

# For PyMongo
pip show pymongo
```

Refer to the MongoDB driver compatibility matrix at docs.mongodb.com/drivers.

## Summary

Use `db.version()` to check the MongoDB binary version and `db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })` to check the FCV. Before any major version upgrade, confirm all replica set members and shards share the same FCV, and upgrade one major version at a time.
