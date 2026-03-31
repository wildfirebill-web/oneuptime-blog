# How to Use --gzip for Compressed Backups in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Backup, Storage

Description: Learn how to use the --gzip flag with mongodump and mongorestore to create compressed MongoDB backups that reduce storage and transfer costs.

---

MongoDB backups can be large. The `--gzip` flag compresses backup files using gzip compression, typically reducing backup size by 70-80% for typical BSON data. This lowers storage costs and speeds up backup transfers over the network.

## Creating a Compressed Backup

Add `--gzip` to a standard `mongodump` command:

```bash
mongodump \
  --db myapp \
  --gzip \
  --out /backup/myapp-compressed
```

Without `--gzip`, each collection produces a `.bson` file. With `--gzip`, the files become `.bson.gz`:

```text
/backup/myapp-compressed/
  myapp/
    orders.bson.gz
    orders.metadata.json.gz
    users.bson.gz
    users.metadata.json.gz
```

## Compressing a Full Backup

Back up all databases with compression:

```bash
mongodump \
  --gzip \
  --out /backup/full-backup-2026-03-31
```

## Using --archive with --gzip

Combine `--archive` and `--gzip` to produce a single compressed file:

```bash
mongodump \
  --db myapp \
  --archive=/backup/myapp-2026-03-31.archive.gz \
  --gzip
```

This is ideal for storing or transferring a complete database backup as one file.

## Checking Compression Ratio

Compare compressed and uncompressed sizes:

```bash
# Uncompressed backup size
mongodump --db myapp --out /tmp/uncompressed
du -sh /tmp/uncompressed/myapp/

# Compressed backup size
mongodump --db myapp --gzip --out /tmp/compressed
du -sh /tmp/compressed/myapp/
```

Sample output:

```text
450M    /tmp/uncompressed/myapp/
 85M    /tmp/compressed/myapp/
```

## Restoring from a Compressed Backup

`mongorestore` automatically detects gzip-compressed files when you add `--gzip`:

```bash
mongorestore \
  --db myapp \
  --gzip \
  /backup/myapp-compressed/myapp/
```

For archive-style compressed backups:

```bash
mongorestore \
  --archive=/backup/myapp-2026-03-31.archive.gz \
  --gzip
```

## Restoring a Single Compressed Collection

```bash
mongorestore \
  --db myapp \
  --collection orders \
  --gzip \
  /backup/myapp-compressed/myapp/orders.bson.gz
```

## Parallel Compression with numParallelCollections

Speed up compressed backups by dumping multiple collections in parallel:

```bash
mongodump \
  --db myapp \
  --gzip \
  --numParallelCollections 4 \
  --out /backup/myapp-parallel
```

## Integrating with Cron

Schedule a nightly compressed backup:

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d)
mongodump \
  --db myapp \
  --gzip \
  --archive="/backup/myapp-${DATE}.archive.gz"

# Remove backups older than 14 days
find /backup -name "myapp-*.archive.gz" -mtime +14 -delete
```

## Summary

The `--gzip` flag compresses `mongodump` output at the file level, typically achieving 70-80% size reduction. Pair it with `--archive` to produce a single portable file. Always include `--gzip` in `mongorestore` when restoring compressed backups, and use `--numParallelCollections` to speed up large database dumps.
