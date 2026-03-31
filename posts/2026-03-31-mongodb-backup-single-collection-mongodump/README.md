# How to Back Up a Single Collection with mongodump in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Backup, Administration

Description: Learn how to use mongodump with the --collection flag to back up a single MongoDB collection and restore it with mongorestore.

---

While `mongodump` can back up an entire database, you can also target a single collection. This is useful for lightweight backups before schema migrations, exporting specific data, or creating point-in-time copies of high-value collections.

## Backing Up a Single Collection

Use the `--db` and `--collection` flags together:

```bash
mongodump \
  --db myapp \
  --collection orders \
  --out /backup/mongodump-2026-03-31
```

This creates the directory structure:

```text
/backup/mongodump-2026-03-31/
  myapp/
    orders.bson
    orders.metadata.json
```

The `.bson` file contains the collection data and the `.metadata.json` file contains index definitions and collection options.

## Backing Up with Authentication

```bash
mongodump \
  --host localhost \
  --port 27017 \
  --username admin \
  --password secret \
  --authenticationDatabase admin \
  --db myapp \
  --collection orders \
  --out /backup/orders-backup
```

## Backing Up to a Specific Archive File

Use `--archive` to write the backup to a single file instead of a directory:

```bash
mongodump \
  --db myapp \
  --collection orders \
  --archive=/backup/orders-2026-03-31.archive
```

## Backing Up with a Query Filter

Export only documents matching a query using `--query`:

```bash
mongodump \
  --db myapp \
  --collection orders \
  --query '{"status": "completed", "createdAt": {"$gte": {"$date": "2026-01-01T00:00:00Z"}}}' \
  --out /backup/completed-orders
```

This backs up only completed orders created since January 2026.

## Restoring a Single Collection

Restore the backed-up collection with `mongorestore`:

```bash
mongorestore \
  --db myapp \
  --collection orders \
  /backup/mongodump-2026-03-31/myapp/orders.bson
```

To restore to a different collection name:

```bash
mongorestore \
  --db myapp \
  --collection orders_restored \
  /backup/mongodump-2026-03-31/myapp/orders.bson
```

## Restoring from an Archive

```bash
mongorestore \
  --archive=/backup/orders-2026-03-31.archive \
  --nsInclude="myapp.orders"
```

## Verifying the Restore

After restoring, confirm document counts match:

```javascript
use myapp

// Count in restored collection
db.orders.countDocuments()

// Cross-check with source if available
db.orders_restored.countDocuments()
```

## Automating Single-Collection Backups

Create a script to back up a collection daily:

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d)
BACKUP_DIR="/backup/orders/${DATE}"

mongodump \
  --db myapp \
  --collection orders \
  --out "${BACKUP_DIR}"

echo "Backup complete: ${BACKUP_DIR}"
```

Save as `/usr/local/bin/backup-orders.sh` and schedule with cron:

```bash
0 2 * * * /usr/local/bin/backup-orders.sh >> /var/log/mongodb-backup.log 2>&1
```

## Summary

Use `mongodump --db myapp --collection orders` to back up a single collection. Add `--query` to filter which documents are exported, and `--archive` to produce a single-file backup. Restore with `mongorestore` specifying the `.bson` file path or archive, and verify document counts after restoring.
