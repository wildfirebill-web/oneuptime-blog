# How to Validate a MongoDB Backup Before Restoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Backup, Validation, Recovery, mongorestore

Description: Validate MongoDB backup integrity before restoring to production by checking BSON files, doing test restores, and verifying document counts.

---

## Why Validate Backups?

A backup is only useful if it can be successfully restored. Corrupted BSON files, incomplete dumps, or permission issues can cause restores to fail at the worst possible moment. Validating backups proactively ensures your recovery point is actually recoverable.

## Check BSON File Integrity

MongoDB stores collection data as BSON files. Use `bsondump` to inspect them:

```bash
bsondump /backup/dump/myapp/users.bson | head -5
```

If the output shows valid JSON documents, the BSON file is readable. A corrupted file will produce errors.

Check all BSON files in a dump:

```bash
find /backup/dump -name "*.bson" -exec sh -c 'echo "Checking: $1"; bsondump "$1" > /dev/null && echo "OK" || echo "FAILED"' _ {} \;
```

## Verify Metadata Files

Each collection dump includes a `.metadata.json` file. Verify it exists alongside its BSON file:

```bash
for bson in /backup/dump/**/*.bson; do
  meta="${bson%.bson}.metadata.json"
  if [ ! -f "$meta" ]; then
    echo "Missing metadata: $meta"
  fi
done
```

## Do a Test Restore to a Separate Instance

The most reliable validation is a full test restore to a separate MongoDB instance:

```bash
# Start a test MongoDB on a different port
mongod --port 27018 --dbpath /tmp/mongo-test --logpath /tmp/mongo-test.log --fork

# Restore the backup
mongorestore \
  --host localhost:27018 \
  --drop \
  /backup/dump/

# Verify document counts
mongosh --port 27018 --eval '
  db.adminCommand({ listDatabases: 1 }).databases.forEach(d => {
    const testDb = db.getSiblingDB(d.name);
    testDb.getCollectionNames().forEach(col => {
      print(d.name + "." + col + ": " + testDb[col].countDocuments());
    });
  });
'
```

## Compare Document Counts with Source

Record counts from the source before backup:

```javascript
// Run on source before backup
db.getCollectionNames().forEach(col => {
  print(col + ":", db[col].countDocuments());
});
```

Compare with restored instance counts. Mismatches indicate incomplete backups.

## Validate Specific Collections

For critical collections, do a deeper check:

```javascript
// Validate collection structure
db.runCommand({ validate: "orders", full: true })
```

This checks index consistency and BSON document validity.

## Verify Archive Integrity

For compressed archive backups, check archive integrity before restoring:

```bash
# For gzip archives
gzip -t /backup/myapp.archive.gz && echo "Archive OK" || echo "Archive CORRUPTED"

# Preview archive contents without restoring
mongorestore \
  --archive=/backup/myapp.archive \
  --gzip \
  --dryRun \
  --verbose
```

## Automate Backup Validation

Add validation to your backup script:

```bash
#!/bin/bash
DUMP_DIR="/backup/dump-$(date +%Y%m%d)"
mongodump --out "$DUMP_DIR" --gzip

# Validate each BSON file
FAILED=0
for bson in "$DUMP_DIR"/**/*.bson; do
  bsondump "$bson" > /dev/null 2>&1 || { echo "FAILED: $bson"; FAILED=1; }
done

if [ $FAILED -eq 1 ]; then
  echo "Backup validation FAILED - alerting on-call"
  curl -X POST https://api.oneuptime.com/webhook/alert -d '{"message":"MongoDB backup validation failed"}'
fi
```

## Summary

Validating MongoDB backups involves checking BSON file readability with `bsondump`, verifying metadata files exist, and doing a test restore to a separate instance. Compare document counts between source and restored data, and use `--dryRun` to preview archive restores. Automate these checks post-backup so issues are caught before they become recovery emergencies.
