# How to Use RIOT for Redis Data Migration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RIOT, Migration, Data Transfer, DevOps

Description: Learn how to use RIOT (Redis Input/Output Tools) to migrate data between Redis instances, import from files, and export to various formats.

---

RIOT (Redis Input/Output Tools) is an open-source toolkit from Redis Labs for migrating data between Redis instances, importing from databases or files, and exporting Redis data. It provides a command-line interface with support for filtering, transformation, and verification.

## Installation

```bash
# Download RIOT
wget https://github.com/redis/riot/releases/latest/download/riot-standalone.zip
unzip riot-standalone.zip
cd riot-standalone/bin

# Verify installation
./riot --version
```

Or using Homebrew on macOS:

```bash
brew install redis/tap/riot
```

## Redis to Redis Migration (riot-redis)

The `replicate` command copies data from one Redis instance to another:

```bash
# Basic replication
riot replicate \
  --source-uri redis://source-host:6379 \
  --target-uri redis://target-host:6379

# With authentication
riot replicate \
  --source-uri redis://:source-password@source-host:6379 \
  --target-uri redis://:target-password@target-host:6379

# With TLS
riot replicate \
  --source-uri rediss://source-host:6379 \
  --target-uri rediss://target-host:6379
```

## Live Replication Mode

For minimal downtime, use the `--live` flag which streams changes after the initial snapshot:

```bash
riot replicate --live \
  --source-uri redis://:password@source-host:6379 \
  --target-uri redis://:password@target-host:6379 \
  --threads 4 \
  --batch 500
```

## Filtering Keys During Migration

```bash
# Migrate only keys matching a pattern
riot replicate \
  --source-uri redis://source-host:6379 \
  --target-uri redis://target-host:6379 \
  --key-pattern "user:*"

# Migrate only specific key types
riot replicate \
  --source-uri redis://source-host:6379 \
  --target-uri redis://target-host:6379 \
  --key-type hash

# Migrate specific database number
riot replicate \
  --source-uri redis://source-host:6379/1 \
  --target-uri redis://target-host:6379/0
```

## Import from CSV File

```bash
# Sample CSV file: users.csv
# id,name,email,age
# 1001,Alice,alice@example.com,30
# 1002,Bob,bob@example.com,25

riot file-import users.csv \
  --uri redis://target-host:6379 \
  --keyspace user \
  --keys id \
  --type hash
```

## Import from JSON File

```bash
# users.json
# [{"id":"1001","name":"Alice","email":"alice@example.com"}]

riot file-import users.json \
  --uri redis://target-host:6379 \
  --keyspace user \
  --keys id \
  --type json
```

## Export Redis Data to File

```bash
# Export to JSON
riot file-export redis-export.json \
  --uri redis://source-host:6379 \
  --key-pattern "user:*"

# Export to CSV (hashes only)
riot file-export export.csv \
  --uri redis://source-host:6379 \
  --key-type hash
```

## Import from a Relational Database

RIOT supports importing directly from SQL databases:

```bash
riot db-import \
  --uri redis://target-host:6379 \
  --db-url "jdbc:postgresql://pg-host:5432/mydb" \
  --db-username dbuser \
  --db-password dbpass \
  --sql "SELECT id, name, email FROM users" \
  --keyspace user \
  --keys id \
  --type hash
```

## Verify Migration Results

RIOT includes a compare command to verify that source and target are in sync:

```bash
riot compare \
  --source-uri redis://:password@source-host:6379 \
  --target-uri redis://:password@target-host:6379

# Sample output:
# Keys matched: 124532
# Keys missing in target: 0
# Keys with different values: 0
# Keys with different TTLs: 3
```

## Monitoring Progress

```bash
# RIOT shows progress by default
# To get more verbose output:
riot replicate \
  --source-uri redis://source-host:6379 \
  --target-uri redis://target-host:6379 \
  --progress log

# Output includes:
# [INFO] Keys/s: 45231 | Keys total: 1245300 | Errors: 0
```

## Summary

RIOT is a versatile Redis migration toolkit that handles Redis-to-Redis replication, file imports/exports, and database imports in a single CLI tool. Use `--live` mode for minimal downtime migrations and always run the `compare` command after migration to verify data integrity.
