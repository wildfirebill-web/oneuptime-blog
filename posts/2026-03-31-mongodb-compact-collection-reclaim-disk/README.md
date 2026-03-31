# How to Compact a Collection to Reclaim Disk Space in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Compact, Disk, Storage, Performance

Description: Learn how to use the compact command in MongoDB to reclaim fragmented disk space after large deletes or updates, and how to minimize production impact.

---

## Overview

When you delete or update large numbers of documents in MongoDB, the freed storage space is not immediately returned to the operating system. Over time, collections become fragmented. The `compact` command defragments a collection and returns unused space to the WiredTiger storage engine.

## Running the compact Command

Connect to the database and run:

```javascript
db.runCommand({ compact: "orders" })
```

Or use `mongosh`:

```bash
mongosh "mongodb://localhost:27017/mydb" --eval 'db.runCommand({ compact: "orders" })'
```

The command rewrites all documents in the collection, reclaiming fragmented space.

## Checking Collection Size Before and After

Check storage statistics before running `compact` to understand how much space might be reclaimed:

```javascript
db.orders.stats()
```

Key fields to look at:

- `storageSize` - total bytes allocated to the collection on disk
- `size` - total size of all documents in bytes
- The difference between `storageSize` and `size` is the fragmented space

After `compact`, run `stats()` again to confirm the reduction.

## Compact with Force on a Primary

By default, `compact` is not allowed on a primary replica set member. Use `force: true` to override this:

```javascript
db.runCommand({ compact: "orders", force: true })
```

Running `compact` on a primary holds a write lock on the collection, so do this during low-traffic windows.

## Running compact on a Secondary First

A safer approach is to run `compact` on secondaries first, then step down the primary and compact it:

```bash
# Connect to each secondary and run compact
mongosh "mongodb://secondary1:27017/mydb" --eval 'db.runCommand({ compact: "orders" })'
mongosh "mongodb://secondary2:27017/mydb" --eval 'db.runCommand({ compact: "orders" })'

# Step down the primary
mongosh "mongodb://primary:27017/admin" --eval 'rs.stepDown()'

# Compact the new secondary (formerly primary)
mongosh "mongodb://primary:27017/mydb" --eval 'db.runCommand({ compact: "orders" })'
```

## Automating Compaction with a Script

```bash
#!/bin/bash
COLLECTIONS=("orders" "events" "logs")
for coll in "${COLLECTIONS[@]}"; do
  echo "Compacting collection: $coll"
  mongosh "mongodb://localhost:27017/mydb" --eval \
    "db.runCommand({ compact: '$coll', force: true })"
done
echo "All collections compacted"
```

## When to Run compact

- After deleting more than 20-30% of documents in a collection
- After bulk updates that significantly changed document sizes
- When `storageSize` greatly exceeds the actual `size` of documents
- During scheduled maintenance windows

## Summary

The `compact` command in MongoDB defragments a collection and reclaims disk space that was freed by deletes and updates. Run it on secondaries before primaries to minimize write lock impact. Always check `db.collection.stats()` before and after to confirm the space reclamation was successful.
