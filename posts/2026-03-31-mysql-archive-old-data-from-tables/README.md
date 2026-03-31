# How to Archive Old Data from MySQL Tables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Archiving, Storage, Partitioning, Data Management

Description: Learn how to archive old data from MySQL tables using pt-archiver, partitioning, and archive tables to keep production tables lean and fast.

---

## Why Archive Old Data?

Large tables accumulate historical data that is rarely queried but still occupies storage and slows down index operations. Archiving moves old rows to a separate table or database, keeping the production table small and fast while retaining historical data for compliance or reporting.

Common archiving candidates:
- Event and audit logs older than 90 days
- Completed orders older than 1 year
- Session and activity records older than 30 days

## Method 1 - pt-archiver (Recommended)

Percona's `pt-archiver` is purpose-built for archiving MySQL rows. It moves rows in small batches to avoid replication lag, locking, and performance impact:

```bash
pt-archiver \
  --source h=primary.db,D=mydb,t=events,u=root,p=secret \
  --dest h=archive.db,D=archive,t=events,u=root,p=secret \
  --where "created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)" \
  --limit 1000 \
  --commit-each \
  --sleep 0.05 \
  --no-check-charset \
  --progress 10000
```

- `--limit 1000` moves 1000 rows per batch
- `--sleep 0.05` waits 50ms between batches to reduce load
- `--commit-each` commits each batch separately

pt-archiver can also delete rows without an archive destination using `--purge` instead of `--dest`.

## Method 2 - Manual Batched INSERT + DELETE

For environments without pt-archiver, implement batched archiving in a script:

```python
import mysql.connector
import time

source = mysql.connector.connect(host="primary.db", user="app",
                                  password="secret", database="mydb")
archive = mysql.connector.connect(host="archive.db", user="app",
                                   password="secret", database="archive")

BATCH_SIZE = 1000
CUTOFF = "2024-01-01"

while True:
    # Get a batch of old rows
    src_cursor = source.cursor(dictionary=True)
    src_cursor.execute(
        "SELECT * FROM events WHERE created_at < %s LIMIT %s",
        (CUTOFF, BATCH_SIZE)
    )
    rows = src_cursor.fetchall()

    if not rows:
        print("Archiving complete.")
        break

    # Insert into archive database
    arch_cursor = archive.cursor()
    arch_cursor.executemany(
        "INSERT IGNORE INTO events (id, user_id, event_type, metadata, created_at) "
        "VALUES (%(id)s, %(user_id)s, %(event_type)s, %(metadata)s, %(created_at)s)",
        rows
    )
    archive.commit()

    # Delete from source
    ids = [r["id"] for r in rows]
    src_cursor.execute(
        f"DELETE FROM events WHERE id IN ({','.join(['%s'] * len(ids))})",
        ids
    )
    source.commit()

    print(f"Archived {len(rows)} rows")
    time.sleep(0.1)
```

## Method 3 - Partition Pruning for Time-Series Tables

If the table uses range partitioning by date, archiving old data is instant - just drop the old partition:

```sql
-- Create a partitioned table
CREATE TABLE events (
  id BIGINT UNSIGNED NOT NULL,
  created_at DATE NOT NULL,
  event_type VARCHAR(50) NOT NULL,
  PRIMARY KEY (id, created_at)
) ENGINE=InnoDB
PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p_2023_q1 VALUES LESS THAN ('2023-04-01'),
  PARTITION p_2023_q2 VALUES LESS THAN ('2023-07-01'),
  PARTITION p_current VALUES LESS THAN MAXVALUE
);

-- Archive Q1 2023 instantly (no row-by-row delete)
ALTER TABLE events DROP PARTITION p_2023_q1;
```

Dropping a partition is a metadata operation - it completes in milliseconds regardless of how many rows the partition contains.

## Setting Up the Archive Table

The archive table on the destination can use the ARCHIVE storage engine for better compression:

```sql
CREATE TABLE events (
  id BIGINT UNSIGNED NOT NULL,
  user_id BIGINT UNSIGNED NOT NULL,
  event_type VARCHAR(50) NOT NULL,
  metadata JSON,
  created_at DATETIME NOT NULL
) ENGINE=ARCHIVE;
```

ARCHIVE tables are write-once, support sequential reads, and compress data automatically - ideal for cold storage.

## Summary

Archive old MySQL data using `pt-archiver` for low-impact incremental archiving, batched INSERT+DELETE scripts for custom logic, or partition drops for instant archiving of time-series tables. The ARCHIVE storage engine on the destination compresses cold data efficiently. Schedule archiving jobs during off-peak hours and monitor replication lag to ensure archiving does not impact replicas.
