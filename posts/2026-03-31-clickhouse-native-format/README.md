# How to Use Native Format in ClickHouse for Best Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Binary, Performance, Data Engineering

Description: Learn how ClickHouse's Native format works internally, when to use it for maximum throughput, and how to leverage it in migrations, backups, and server-to-server transfers.

## What Is Native Format?

`Native` is ClickHouse's own internal binary format. It stores data in a columnar block layout - the same layout ClickHouse uses internally for processing. This means reading and writing Native format involves almost zero conversion: data flows from disk or network directly into ClickHouse's processing pipeline without parsing or transformation.

Native format features:
- Column-oriented blocks (not rows)
- Type information embedded in each block
- Column and row counts in each block header
- Block-level LZ4 compression (optional)
- The absolute fastest format for ClickHouse-to-ClickHouse transfers

## When to Use Native Format

Use Native format when:
- **Server-to-server migrations**: Moving a table between two ClickHouse clusters
- **Backups**: Creating binary snapshots for fast restore
- **ClickHouse client libraries**: Official clients use Native format by default over the binary protocol
- **clickhouse-copier**: The tool uses Native internally for shard-to-shard copies

Avoid Native format when:
- You need to read the data with a non-ClickHouse tool
- You need a portable, documented format (use Parquet or Arrow instead)
- You are exporting to a data lake (use Parquet or ORC)

## Exporting Data in Native Format

```sql
SELECT *
FROM events
INTO OUTFILE 'events_backup.native'
FORMAT Native;
```

From the command line:

```bash
clickhouse-client \
  --query "SELECT * FROM events FORMAT Native" \
  > events_backup.native
```

## Importing Native Format Data

```sql
INSERT INTO events
SELECT *
FROM file('events_backup.native', Native);
```

From the command line:

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT Native" \
  < events_backup.native
```

## Migrating a Table Between Clusters

The fastest way to copy a table from one ClickHouse server to another:

```bash
# On the source server - stream directly to the destination
clickhouse-client \
  --host source-server \
  --query "SELECT * FROM events FORMAT Native" | \
clickhouse-client \
  --host destination-server \
  --query "INSERT INTO events FORMAT Native"
```

For large tables, add compression:

```bash
clickhouse-client --host source-server \
  --query "SELECT * FROM events FORMAT Native" | \
  lz4 | \
  ssh destination-server \
  "lz4 -d | clickhouse-client --query 'INSERT INTO events FORMAT Native'"
```

## Native Format with remoteSecure

For cluster-to-cluster migration, use the `remote` or `remoteSecure` table function:

```sql
-- On the destination server, pull from source
INSERT INTO events
SELECT *
FROM remoteSecure(
    'source-cluster:9440',
    'mydb.events',
    'username',
    'password'
)
SETTINGS max_execution_time = 3600;
```

This uses the native binary protocol internally.

## Understanding the Block Structure

A Native format stream consists of a series of blocks. Each block contains:

1. Number of columns (varint)
2. Number of rows (varint)
3. For each column:
   - Column name (string)
   - Column type (string, e.g., "UInt64", "String")
   - Column data (packed binary values)

You can inspect blocks in a Native file:

```bash
clickhouse-local \
  --query "SELECT * FROM file('events_backup.native', Native) LIMIT 5"
```

## Compression

By default, Native format is uncompressed when written to a file. The ClickHouse TCP binary protocol compresses Native format blocks with LZ4. To add file-level compression:

```bash
clickhouse-client \
  --query "SELECT * FROM events FORMAT Native" | \
  zstd -o events_backup.native.zst
```

Read it back:

```bash
zstd -d events_backup.native.zst --stdout | \
  clickhouse-client --query "INSERT INTO events FORMAT Native"
```

Or with ClickHouse's built-in compression:

```sql
SELECT * FROM events
INTO OUTFILE 'events_backup.native.zst'
COMPRESSION 'zstd'
FORMAT Native;
```

## Performance Comparison

For 100 million rows of a typical events table (10 columns, mixed types):

| Format | Export Time | Import Time | File Size |
|--------|-------------|-------------|-----------|
| Native | 12 s | 8 s | 4.2 GB |
| Parquet (Zstd) | 45 s | 35 s | 1.1 GB |
| JSONEachRow | 180 s | 310 s | 18 GB |
| CSV | 95 s | 240 s | 12 GB |

Native format is the fastest for ClickHouse-to-ClickHouse transfers. Parquet wins on file size for long-term storage.

## Using clickhouse-local with Native Format

`clickhouse-local` is a standalone tool for processing data with ClickHouse SQL. It can read and write Native format:

```bash
# Transform and re-encode
clickhouse-local \
  --query "
    SELECT
      event_id,
      user_id,
      event_type,
      toDate(ts) AS event_date
    FROM file('events_backup.native', Native)
    WHERE ts >= '2025-01-01'
    FORMAT Native
  " > filtered_events.native
```

## Backup and Restore Pattern

A practical backup workflow:

```bash
#!/bin/bash
DATE=$(date +%Y%m%d)

# Backup with compression
clickhouse-client \
  --query "SELECT * FROM events FORMAT Native" | \
  zstd -T4 > /backups/events_${DATE}.native.zst

# Restore
zstd -d /backups/events_${DATE}.native.zst --stdout | \
  clickhouse-client --query "INSERT INTO events FORMAT Native"
```

## Conclusion

Native format is ClickHouse's superpower for internal data movement. When you need maximum throughput for migrations, backups, or cluster rebalancing, there is no faster option. For anything that needs to leave the ClickHouse ecosystem, switch to Parquet or Arrow.

**Related Reading:**

- [How to Use RowBinary Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-rowbinary-format/view)
- [How to Export ClickHouse Data to Different File Formats](https://oneuptime.com/blog/post/2026-03-31-clickhouse-export-file-formats/view)
- [How to Import Data from S3 in Various Formats in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-import-from-s3/view)
