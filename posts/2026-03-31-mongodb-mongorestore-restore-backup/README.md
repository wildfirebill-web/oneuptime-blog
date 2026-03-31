# How to Use mongorestore to Restore a MongoDB Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, mongorestore, Backup, Restore, Recovery

Description: Use mongorestore to restore MongoDB backups created by mongodump, with options for full, partial, and selective collection restores.

---

## What Is mongorestore?

`mongorestore` is the companion tool to `mongodump`. It reads BSON data files created by `mongodump` and inserts them into a MongoDB instance. It supports restoring full databases, individual collections, and filtered subsets.

## Basic Restore Syntax

```bash
mongorestore \
  --host localhost:27017 \
  --username admin \
  --password secret \
  --authenticationDatabase admin \
  /path/to/dump/
```

## Restore a Full Backup

If `mongodump` was run without specifying a database, it dumps all databases into a directory. Restore all of them:

```bash
mongorestore \
  --host localhost:27017 \
  -u admin -p secret \
  --authenticationDatabase admin \
  --drop \
  /backup/dump/
```

The `--drop` flag drops existing collections before restoring, preventing duplicate document errors.

## Restore a Single Database

```bash
mongorestore \
  --host localhost:27017 \
  -u admin -p secret \
  --authenticationDatabase admin \
  --db myapp \
  --drop \
  /backup/dump/myapp/
```

## Restore a Single Collection

```bash
mongorestore \
  --host localhost:27017 \
  -u admin -p secret \
  --authenticationDatabase admin \
  --db myapp \
  --collection users \
  --drop \
  /backup/dump/myapp/users.bson
```

## Restore from a Compressed Archive

If you used `mongodump --archive --gzip`:

```bash
mongorestore \
  --host localhost:27017 \
  -u admin -p secret \
  --authenticationDatabase admin \
  --gzip \
  --archive=/backup/myapp-20260101.archive \
  --drop
```

Or read from stdin:

```bash
cat /backup/myapp.archive | mongorestore \
  --archive \
  --gzip \
  --drop
```

## Restore to a Different Database

Use `--nsFrom` and `--nsTo` to restore into a different namespace:

```bash
mongorestore \
  --nsFrom "myapp.*" \
  --nsTo "myapp_restored.*" \
  /backup/dump/
```

## Parallel Restore for Speed

Use `--numParallelCollections` to speed up large restores:

```bash
mongorestore \
  --numParallelCollections 4 \
  --numInsertionWorkersPerCollection 2 \
  /backup/dump/
```

## Verifying the Restore

After restoring, verify document counts:

```javascript
use myapp
db.getCollectionNames().forEach(col => {
  print(col, ":", db[col].countDocuments());
});
```

## Summary

`mongorestore` is straightforward for full or partial restores from `mongodump` outputs. Use `--drop` to avoid duplicates, `--archive` for single-file backups, and `--nsFrom/--nsTo` to redirect restores to a different database. Always verify document counts after restoring to confirm the operation succeeded.
