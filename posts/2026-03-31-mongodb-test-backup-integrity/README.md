# How to Test Backup Integrity for MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Backup, Integrity, Testing, Recovery

Description: Test MongoDB backup integrity using restore drills, document count comparisons, and checksums to ensure backups are reliable for recovery.

---

## The Importance of Backup Testing

A backup that has never been tested is not a real backup. Corrupted files, incomplete transfers, and version incompatibilities are often discovered only when you desperately need to restore. Regular backup integrity testing is a non-negotiable part of any database operations strategy.

## Test 1: BSON File Checksums

Calculate checksums immediately after creating a backup, then verify before restoring:

```bash
# After backup
find /backup/dump -name "*.bson" -exec md5sum {} \; > /backup/dump/checksums.md5

# Before restore - verify nothing changed
md5sum -c /backup/dump/checksums.md5
```

## Test 2: bsondump Validation

Parse each BSON file to confirm it's readable:

```bash
for bson in /backup/dump/**/*.bson; do
  count=$(bsondump "$bson" 2>/dev/null | wc -l)
  echo "$bson: $count documents"
done
```

Zero documents in a non-empty collection indicates corruption.

## Test 3: Full Restore Drill

Spin up a temporary MongoDB instance and perform a full restore:

```bash
# Create test environment
docker run -d \
  --name mongo-test \
  -p 27099:27017 \
  mongo:7.0

# Wait for startup
sleep 5

# Restore backup
mongorestore \
  --host localhost:27099 \
  --drop \
  /backup/dump/

# Verify counts
mongosh --port 27099 --eval '
  db.adminCommand({ listDatabases: 1 }).databases.forEach(d => {
    if (d.name === "admin" || d.name === "config" || d.name === "local") return;
    const testDb = db.getSiblingDB(d.name);
    print("DB:", d.name, "Collections:", testDb.getCollectionNames().length);
  });
'

# Cleanup
docker stop mongo-test && docker rm mongo-test
```

## Test 4: Spot-Check Data Correctness

Verify that critical records are intact after restore:

```javascript
// On restored instance
use myapp

// Check a known document
db.users.findOne({ email: "test@example.com" })

// Verify index existence
db.users.getIndexes()

// Check referential integrity
db.orders.aggregate([
  { $lookup: {
    from: "users",
    localField: "userId",
    foreignField: "_id",
    as: "user"
  }},
  { $match: { user: { $size: 0 } } },
  { $count: "orphaned_orders" }
])
```

## Test 5: Collection Validation

Run MongoDB's built-in validation on restored collections:

```javascript
db.getCollectionNames().forEach(col => {
  const result = db.runCommand({ validate: col, full: true });
  if (!result.valid) {
    print("INVALID:", col, JSON.stringify(result.errors));
  } else {
    print("OK:", col);
  }
});
```

## Automating Backup Tests

Schedule weekly backup tests with a script:

```bash
#!/bin/bash
set -e
BACKUP_DIR="/backup/dump-latest"
LOG="/var/log/mongo-backup-test.log"

echo "$(date) - Starting backup integrity test" >> $LOG

# Start test instance
docker run -d --name mongo-backup-test -p 27099:27017 mongo:7.0
sleep 5

# Restore
mongorestore --host localhost:27099 --drop "$BACKUP_DIR" >> $LOG 2>&1

# Count check
RESULT=$(mongosh --port 27099 --quiet --eval '
  let total = 0;
  db.adminCommand({ listDatabases: 1 }).databases.forEach(d => {
    if (["admin","config","local"].includes(d.name)) return;
    db.getSiblingDB(d.name).getCollectionNames().forEach(c => {
      total += db.getSiblingDB(d.name)[c].countDocuments();
    });
  });
  total;
')

echo "$(date) - Restored $RESULT documents" >> $LOG

# Cleanup
docker stop mongo-backup-test && docker rm mongo-backup-test

echo "$(date) - Backup test PASSED" >> $LOG
```

## Summary

Testing MongoDB backup integrity requires more than just checking that files exist. Use `bsondump` for BSON validation, checksums for transfer integrity, full restore drills to temporary instances, and spot-checks for data correctness. Automate these tests weekly and alert on failures so you always know your backups are reliable before you need them.
