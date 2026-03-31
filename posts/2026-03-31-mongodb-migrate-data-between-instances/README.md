# How to Migrate Data Between MongoDB Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Migration, Mongodump, mongorestore, Database

Description: Learn how to migrate data between MongoDB instances using mongodump and mongorestore, including full cluster migrations, selective collection copies, and live sync strategies.

---

## Overview

Migrating data between MongoDB instances is a common task when moving between environments, upgrading hardware, or switching cloud providers. MongoDB provides `mongodump` and `mongorestore` for offline migrations, and `mongomirror` or change streams for live migrations.

## Full Database Migration with mongodump and mongorestore

Dump all databases from the source:

```bash
mongodump \
  --uri "mongodb://source-host:27017" \
  --out /backup/dump
```

Restore to the destination:

```bash
mongorestore \
  --uri "mongodb://dest-host:27017" \
  --dir /backup/dump
```

## Migrating a Single Database

```bash
mongodump \
  --uri "mongodb://source-host:27017/mydb" \
  --out /backup/mydb_dump

mongorestore \
  --uri "mongodb://dest-host:27017" \
  --db mydb \
  /backup/mydb_dump/mydb
```

## Migrating a Single Collection

```bash
mongodump \
  --uri "mongodb://source-host:27017/mydb" \
  --collection orders \
  --out /backup

mongorestore \
  --uri "mongodb://dest-host:27017" \
  --db mydb \
  --collection orders \
  /backup/mydb/orders.bson
```

## Piping Directly Between Instances

Skip the intermediate file by piping dump output directly to mongorestore:

```bash
mongodump \
  --uri "mongodb://source-host:27017/mydb" \
  --archive | \
mongorestore \
  --uri "mongodb://dest-host:27017" \
  --archive \
  --nsFrom "mydb.*" \
  --nsTo "mydb.*"
```

## Migrating Replica Sets

When migrating to a replica set, connect to the primary:

```bash
mongodump \
  --uri "mongodb://primary:27017/mydb?replicaSet=rs0" \
  --out /backup/dump

mongorestore \
  --uri "mongodb://new-primary:27017/mydb?replicaSet=rs1" \
  --dir /backup/dump
```

## Preserving Indexes

By default, `mongorestore` rebuilds indexes after restoring data. To skip index rebuilding (useful for large collections when you plan to recreate indexes separately):

```bash
mongorestore \
  --uri "mongodb://dest-host:27017" \
  --noIndexRestore \
  --dir /backup/dump
```

Then create indexes manually after verifying the data.

## Verifying the Migration

After restoring, verify document counts match:

```javascript
// On source
db.orders.countDocuments()

// On destination
db.orders.countDocuments()
```

Also compare checksums for critical collections:

```javascript
db.runCommand({ dbHash: 1, collections: ["orders", "customers"] })
```

## Summary

Use `mongodump` and `mongorestore` for reliable offline migrations between MongoDB instances. Pipe the archive directly between hosts to avoid disk space requirements for intermediate files. Always verify document counts and optionally run `dbHash` to confirm data integrity after the migration is complete.
