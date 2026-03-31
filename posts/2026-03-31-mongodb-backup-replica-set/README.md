# How to Back Up a Replica Set in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Backup, Replica Set

Description: Learn how to back up a MongoDB replica set using mongodump against a secondary to minimize primary load and ensure a consistent database backup.

---

Backing up a MongoDB replica set correctly means targeting a secondary member so you do not add backup load to the primary. This guide covers using `mongodump` against a replica set and best practices for consistent, reliable backups.

## Why Back Up from a Secondary

Running `mongodump` against the primary adds read load during the backup window. Targeting a secondary keeps the primary available for application traffic. MongoDB replica sets replicate all data to secondaries, so a secondary backup is identical in content.

## Connecting to the Replica Set

Specify the replica set connection string so the driver can discover all members and handle failovers:

```bash
mongodump \
  --host rs0/mongo1:27017,mongo2:27017,mongo3:27017 \
  --readPreference secondary \
  --out /backup/replica-backup-2026-03-31
```

The `--readPreference secondary` flag directs `mongodump` to read from a secondary member.

## Full Backup with Authentication

```bash
mongodump \
  --host rs0/mongo1:27017,mongo2:27017,mongo3:27017 \
  --readPreference secondary \
  --username backup_user \
  --password secret \
  --authenticationDatabase admin \
  --out /backup/replica-full-backup
```

## Backing Up Specific Databases

```bash
mongodump \
  --host rs0/mongo1:27017,mongo2:27017,mongo3:27017 \
  --readPreference secondary \
  --db myapp \
  --out /backup/myapp-backup
```

## Compressed Backup from a Replica Set

```bash
mongodump \
  --host rs0/mongo1:27017,mongo2:27017,mongo3:27017 \
  --readPreference secondary \
  --gzip \
  --archive=/backup/replica-2026-03-31.archive.gz
```

## Using a Dedicated Backup Secondary

For large production replica sets, add a hidden secondary dedicated to backups:

```javascript
// Add a hidden secondary with priority 0
rs.add({
  host: "backup-host:27017",
  hidden: true,
  priority: 0,
  votes: 0
})
```

Then run backups exclusively against this member:

```bash
mongodump \
  --host backup-host:27017 \
  --out /backup/full-backup
```

## Ensuring Backup Consistency

`mongodump` captures a point-in-time snapshot using the oplog. To make the backup consistent and restorable to a specific point in time, include the oplog dump:

```bash
mongodump \
  --host rs0/mongo1:27017,mongo2:27017,mongo3:27017 \
  --readPreference secondary \
  --oplog \
  --out /backup/consistent-backup
```

The `--oplog` flag captures the oplog entries that occurred during the dump, enabling point-in-time restore with `mongorestore --oplogReplay`.

## Restoring a Replica Set Backup

Restore to a standalone instance first, then convert to a replica set:

```bash
mongorestore \
  --host localhost:27017 \
  --drop \
  --oplogReplay \
  /backup/consistent-backup/
```

To restore the oplog for point-in-time recovery:

```bash
mongorestore \
  --oplogReplay \
  --oplogLimit 1743379200:1 \
  /backup/consistent-backup/
```

The `--oplogLimit` timestamp is in `seconds:ordinal` format.

## Verifying the Backup

After backup, verify it is complete:

```bash
mongorestore \
  --host localhost:27018 \
  --dryRun \
  /backup/consistent-backup/
```

The `--dryRun` flag checks the backup files without actually restoring.

## Summary

Back up a MongoDB replica set by running `mongodump` with `--readPreference secondary` to avoid impacting the primary. Use `--oplog` to capture a consistent point-in-time snapshot, and restore with `--oplogReplay` when consistency is critical. For best results, use a dedicated hidden secondary member for backups.
