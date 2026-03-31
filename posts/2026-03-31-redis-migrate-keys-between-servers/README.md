# How to Use MIGRATE in Redis to Transfer Keys Between Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, MIGRATE, Key Transfer, Data Migration, Cluster

Description: Learn how to use the MIGRATE command in Redis to atomically move keys from one Redis server to another, useful for resharding and live data migrations.

---

## What Is MIGRATE?

`MIGRATE` transfers one or more keys from the current Redis server to a destination server atomically. The command combines `DUMP`, network transfer, `RESTORE`, and optionally `DEL` into a single operation.

This is the primary mechanism used by Redis Cluster for resharding, and it can also be used manually for data migration tasks.

## Basic Syntax

```text
MIGRATE host port key|"" destination-db timeout [COPY] [REPLACE] [AUTH password] [AUTH2 username password] [KEYS key [key ...]]
```

## Migrating a Single Key

```bash
redis-cli MIGRATE 192.168.1.20 6379 mykey 0 5000
```

This:
1. Serializes `mykey` from the current server
2. Sends it to `192.168.1.20:6379` database 0
3. Has a 5000ms timeout
4. Deletes `mykey` from the source after success

## Migrating Without Deleting (COPY)

```bash
redis-cli MIGRATE 192.168.1.20 6379 mykey 0 5000 COPY
```

`COPY` keeps the key on the source server after migration.

## Replacing Existing Keys (REPLACE)

If the key already exists on the destination:

```bash
redis-cli MIGRATE 192.168.1.20 6379 mykey 0 5000 REPLACE
```

Without `REPLACE`, migration fails with an error if the key already exists.

## Migrating Multiple Keys at Once

```bash
redis-cli MIGRATE 192.168.1.20 6379 "" 0 5000 KEYS key1 key2 key3
```

When using `KEYS`, pass an empty string `""` as the key name argument. This is more efficient than migrating keys one at a time.

## Migrating with Authentication

```bash
redis-cli MIGRATE 192.168.1.20 6379 mykey 0 5000 AUTH mypassword
```

For Redis 6.0+ with ACL users:

```bash
redis-cli MIGRATE 192.168.1.20 6379 mykey 0 5000 AUTH2 admin adminpassword
```

## Batch Migration Script

```bash
#!/bin/bash
DEST_HOST=192.168.1.20
DEST_PORT=6379
TIMEOUT=5000

redis-cli --scan --pattern "user:*" | while read KEY; do
  redis-cli MIGRATE "$DEST_HOST" "$DEST_PORT" "$KEY" 0 "$TIMEOUT" COPY
  echo "Migrated: $KEY"
done
```

## How Atomicity Works

`MIGRATE` is atomic from the destination's perspective. The key appears on the destination only after successful transfer. If the source deletes the key and the connection drops before the destination confirms, Redis has mechanisms to avoid data loss - but you should always use `COPY` first in critical migrations and verify before deleting.

## Checking Migration Success

```bash
# On destination
redis-cli -h 192.168.1.20 -p 6379 EXISTS mykey
```

## Limitations

- Keys migrated to Cluster must map to the same slot on the destination
- Very large keys can hit the timeout; increase it for large values
- `MIGRATE` uses a blocking connection to the destination

## Summary

`MIGRATE` is Redis's built-in key transfer command, moving data between servers atomically. Use it for manual data migrations, cluster resharding prep, and moving hot keys between instances. The `COPY` and `KEYS` options make it flexible for both safe testing and bulk operations.
