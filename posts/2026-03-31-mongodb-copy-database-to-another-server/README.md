# How to Copy a MongoDB Database to Another Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Migration, Administration

Description: Learn how to copy a MongoDB database to another server using mongodump, mongorestore, and direct filesystem copy methods with minimal downtime.

---

Copying a MongoDB database to another server is a common task for migrations, staging environment refreshes, and disaster recovery testing. This guide covers the main approaches.

## Method 1: mongodump and mongorestore (Recommended)

This is the most portable and widely used approach.

**On the source server - dump the database:**

```bash
mongodump \
  --host localhost \
  --port 27017 \
  --username admin \
  --password secret \
  --authenticationDatabase admin \
  --db myapp \
  --out /tmp/myapp-dump
```

**Transfer the dump to the destination server:**

```bash
rsync -avz /tmp/myapp-dump/ user@destination-host:/tmp/myapp-dump/
```

**On the destination server - restore the database:**

```bash
mongorestore \
  --host localhost \
  --port 27017 \
  --username admin \
  --password secret \
  --authenticationDatabase admin \
  --db myapp \
  /tmp/myapp-dump/myapp/
```

## Method 2: mongoexport and mongoimport (JSON)

Use this approach when you need a human-readable format or cross-version compatibility:

**Export on the source:**

```bash
mongoexport \
  --db myapp \
  --collection users \
  --out /tmp/users.json

mongoexport \
  --db myapp \
  --collection orders \
  --out /tmp/orders.json
```

**Import on the destination:**

```bash
mongoimport \
  --db myapp \
  --collection users \
  --file /tmp/users.json

mongoimport \
  --db myapp \
  --collection orders \
  --file /tmp/orders.json
```

Note: `mongoexport`/`mongoimport` does not preserve indexes - recreate them manually.

## Method 3: Filesystem Snapshot (Fastest for Large Databases)

For large databases with minimal downtime requirements:

**Step 1 - Lock writes on the source:**

```javascript
db.fsyncLock()
```

**Step 2 - Take a filesystem snapshot (LVM example):**

```bash
sudo lvcreate --snapshot --name mongodb-snap --size 20G /dev/vg0/mongodb-data
```

**Step 3 - Unlock MongoDB:**

```javascript
db.fsyncUnlock()
```

**Step 4 - Mount the snapshot and transfer:**

```bash
sudo mount -o ro /dev/vg0/mongodb-snap /mnt/mongodb-snap
rsync -avz /mnt/mongodb-snap/ user@destination-host:/var/lib/mongodb/
sudo umount /mnt/mongodb-snap
sudo lvremove -f /dev/vg0/mongodb-snap
```

**Step 5 - Start MongoDB on the destination:**

```bash
sudo chown -R mongodb:mongodb /var/lib/mongodb
sudo systemctl start mongod
```

## Verifying the Copy

On the destination server, verify document counts match:

```javascript
use myapp
db.users.countDocuments()
db.orders.countDocuments()
db.stats()
```

Compare with the source server output.

## Handling Authentication

If the destination server uses different credentials, create the application user after restoring:

```javascript
use myapp
db.createUser({
  user: "appuser",
  pwd: "newpassword",
  roles: [{ role: "readWrite", db: "myapp" }]
})
```

## Summary

The recommended way to copy a MongoDB database to another server is using `mongodump` and `mongorestore`, which preserves indexes and supports authentication. For large databases, filesystem snapshots provide faster transfer with minimal write locks. Always verify document counts after copying before decommissioning the source.
