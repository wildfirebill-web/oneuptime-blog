# How to Use MIGRATE in Redis to Transfer Keys Between Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Migration, Cluster

Description: Learn how to use the Redis MIGRATE command to atomically transfer keys from one Redis server to another for live migrations and cluster resharding.

---

The `MIGRATE` command in Redis transfers one or more keys from the source server to a destination server atomically. The key is removed from the source once the destination confirms receipt. This makes it suitable for live data migrations, cluster resharding, and moving data between Redis instances without downtime.

## Basic Syntax

```text
MIGRATE host port key|"" destination-db timeout [COPY] [REPLACE] [AUTH password] [AUTH2 username password] [KEYS key [key ...]]
```

- `host` - Target server hostname or IP.
- `port` - Target server port.
- `key` - The key to migrate (use `""` when using the KEYS option).
- `destination-db` - The database index on the target server.
- `timeout` - Milliseconds to wait for the operation to complete.
- `COPY` - Do not remove the key from the source after migration.
- `REPLACE` - Overwrite the key on the destination if it already exists.
- `KEYS` - Migrate multiple keys in a single call.

## Migrating a Single Key

```bash
# Migrate key "session:abc" from current server to 192.168.1.20:6379 db 0
MIGRATE 192.168.1.20 6379 session:abc 0 5000
```

On success, Redis returns `OK` and removes the key from the source.

## Migrating Multiple Keys

```bash
MIGRATE 192.168.1.20 6379 "" 0 5000 KEYS user:1 user:2 user:3
```

Using `""` as the key name with the `KEYS` option lets you migrate several keys in one round trip.

## Using COPY to Keep Source Key

```bash
MIGRATE 192.168.1.20 6379 config:global 0 5000 COPY
```

With `COPY`, the key remains on the source. This is useful for replicating data rather than moving it.

## Using REPLACE to Overwrite on Destination

```bash
MIGRATE 192.168.1.20 6379 session:abc 0 5000 REPLACE
```

Without `REPLACE`, MIGRATE returns an error if the key already exists at the destination.

## Migrating with Authentication

If the destination server requires a password:

```bash
MIGRATE 192.168.1.20 6379 session:abc 0 5000 AUTH mypassword
```

For Redis 6+ ACL users:

```bash
MIGRATE 192.168.1.20 6379 session:abc 0 5000 AUTH2 myuser mypassword
```

## Scripted Migration with redis-cli

```bash
#!/bin/bash
SOURCE="127.0.0.1:6379"
DEST_HOST="192.168.1.20"
DEST_PORT="6380"

# Get all keys matching a pattern
KEYS=$(redis-cli -h 127.0.0.1 -p 6379 KEYS "session:*")

for KEY in $KEYS; do
  redis-cli -h 127.0.0.1 -p 6379 MIGRATE $DEST_HOST $DEST_PORT "$KEY" 0 5000 REPLACE
  echo "Migrated: $KEY"
done
```

## Error Handling

MIGRATE can return several errors:

```text
IOERR      - Network error during transfer
BUSYKEY    - Key exists on destination (use REPLACE to override)
NOKEY      - Key does not exist on source
```

Always check the return value in scripts before assuming success.

## MIGRATE in Redis Cluster

During cluster resharding, Redis automatically uses MIGRATE internally to move keys between slot owners. The `redis-cli --cluster reshard` command orchestrates this process:

```bash
redis-cli --cluster reshard 127.0.0.1:7001
```

You can also use MIGRATE manually when building custom migration tooling or moving keys from a standalone Redis to a cluster.

## Summary

The Redis MIGRATE command provides atomic key transfer between servers, making it the foundation for live migrations and cluster resharding. Use COPY to preserve the source key, REPLACE to overwrite conflicts, and the KEYS option to batch multiple keys in a single call.
