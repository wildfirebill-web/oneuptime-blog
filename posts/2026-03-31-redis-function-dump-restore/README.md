# How to Use FUNCTION DUMP and RESTORE in Redis for Function Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, FUNCTION DUMP, FUNCTION RESTORE, Function, Backup

Description: Learn how to use FUNCTION DUMP and FUNCTION RESTORE in Redis to serialize all function libraries to binary and restore them on another instance.

---

## What are FUNCTION DUMP and FUNCTION RESTORE

FUNCTION DUMP serializes all currently loaded function libraries into a binary RDB payload. FUNCTION RESTORE takes that binary payload and loads it into a Redis instance, restoring all libraries. Together they provide a portable backup and migration mechanism for Redis Functions.

```redis
FUNCTION DUMP
FUNCTION RESTORE serialized-value [FLUSH | APPEND | REPLACE]
```

```mermaid
flowchart LR
    A[Source Redis\nFunction libraries loaded] --> B[FUNCTION DUMP]
    B --> C[Binary RDB payload]
    C --> D[Transfer to target instance]
    D --> E[FUNCTION RESTORE payload]
    E --> F[Target Redis\nFunction libraries restored]
```

## FUNCTION DUMP

FUNCTION DUMP returns a bulk string containing all function libraries serialized in RDB format. The output is binary and not human-readable.

```redis
FUNCTION DUMP
-- Returns: binary RDB payload (shown as escaped bytes)
-- "\xf6\x00\x03..." etc.
```

### Capturing the dump in redis-cli

```bash
redis-cli --no-auth-warning FUNCTION DUMP > functions.rdb
```

This saves the binary payload to a file.

## FUNCTION RESTORE

FUNCTION RESTORE loads the binary payload produced by FUNCTION DUMP.

```redis
FUNCTION RESTORE <serialized-value>
```

### Conflict policy options

| Policy | Behavior |
|---|---|
| `FLUSH` | Delete all existing libraries before restoring |
| `APPEND` | Add libraries from the dump without deleting existing ones; error on name conflict |
| `REPLACE` | Replace existing libraries with the same name from the dump |

Default behavior (no policy) is equivalent to APPEND.

```redis
-- Restore with FLUSH: replace everything
FUNCTION RESTORE <payload> FLUSH

-- Restore with APPEND: add new libraries only
FUNCTION RESTORE <payload> APPEND

-- Restore with REPLACE: overwrite matching library names
FUNCTION RESTORE <payload> REPLACE
```

## Practical Migration Workflow

### Step 1: Dump from source

```bash
redis-cli -h source-host -p 6379 FUNCTION DUMP > functions_backup.rdb
```

### Step 2: Restore on target

```bash
redis-cli -h target-host -p 6379 FUNCTION RESTORE "$(cat functions_backup.rdb)"
```

For binary-safe transfer, use the raw mode:

```bash
redis-cli -h target-host --pipe-mode < functions_backup.rdb
```

Or use a Redis client that handles binary data natively.

### Step 3: Verify

```redis
FUNCTION LIST
```

```mermaid
sequenceDiagram
    participant S as Source Redis
    participant F as File / Transfer
    participant T as Target Redis

    S->>F: FUNCTION DUMP -> binary payload
    F->>T: Transfer binary payload
    T->>T: FUNCTION RESTORE payload REPLACE
    T-->>T: All libraries loaded
    T->>T: FUNCTION LIST -> verify
```

## Full Backup Example Using redis-cli

```bash
# Backup all functions from production
redis-cli -h prod-redis FUNCTION DUMP | base64 > functions_b64.txt

# Restore to staging
cat functions_b64.txt | base64 -d | redis-cli -h staging-redis --pipe FUNCTION RESTORE - REPLACE
```

## Difference from RDB Persistence

Function libraries are included in the regular RDB snapshot automatically. FUNCTION DUMP provides a targeted, on-demand export containing only function data, useful when:
- You want to migrate only functions, not all data
- You need to copy functions to a new empty instance
- You want a versioned backup of your function libraries separate from data backups

| Method | Includes | Portable | On-demand |
|---|---|---|---|
| RDB snapshot | All data + functions | Yes | No (scheduled) |
| FUNCTION DUMP | Functions only | Yes | Yes |

## Error Cases

```redis
-- Attempting to restore a library that already exists without REPLACE
FUNCTION RESTORE <payload>
-- ERR: Library already exists

-- Corrupted or wrong-version payload
FUNCTION RESTORE invalidbytes
-- ERR: DUMP payload version or checksum are wrong
```

## Summary

FUNCTION DUMP serializes all Redis function libraries to a portable binary format, and FUNCTION RESTORE loads them back on any compatible Redis instance. Use the FLUSH, APPEND, or REPLACE policy to control how conflicts are handled during restore. This pair of commands is the standard way to migrate, backup, or deploy function libraries across Redis environments without copying the full dataset.
