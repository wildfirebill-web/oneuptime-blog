# How to Purge Old Data from MySQL Efficiently

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Management, Performance, Delete, Partitioning

Description: Learn how to purge old data from MySQL tables efficiently using small batched deletes, partition drops, and scheduled events to avoid locking and replication lag.

---

## Why Large Deletes Are Dangerous

Running a single `DELETE FROM events WHERE created_at < '2023-01-01'` on a table with 100 million matching rows causes multiple problems:
- Acquires row locks on all affected rows for the duration of the transaction
- Generates a large undo log that bloats the InnoDB undo tablespace
- Creates a massive binary log event that causes replication lag on replicas
- Blocks other queries waiting for the same rows

The correct approach is always small, batched deletes with pauses between batches.

## Method 1 - Batched Delete Loop

Delete rows in small chunks using a loop. A batch of 1,000-5,000 rows is typically safe:

```sql
-- Repeat until no rows remain
DELETE FROM events
WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)
LIMIT 2000;
```

Track progress with:

```sql
SELECT ROW_COUNT();
```

Automate as a stored procedure:

```sql
DELIMITER //
CREATE PROCEDURE purge_old_events()
BEGIN
  DECLARE affected INT DEFAULT 1;
  WHILE affected > 0 DO
    DELETE FROM events
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)
    LIMIT 2000;
    SET affected = ROW_COUNT();
    DO SLEEP(0.05);
  END WHILE;
END //
DELIMITER ;
```

Call it during off-peak hours:

```sql
CALL purge_old_events();
```

## Method 2 - Partition Drop (Fastest)

If the table is partitioned by date range, dropping a partition is a metadata-only operation - instantaneous regardless of row count:

```sql
-- Partition by quarter
ALTER TABLE events
PARTITION BY RANGE (TO_DAYS(created_at)) (
  PARTITION p_q1_2024 VALUES LESS THAN (TO_DAYS('2024-04-01')),
  PARTITION p_q2_2024 VALUES LESS THAN (TO_DAYS('2024-07-01')),
  PARTITION p_q3_2024 VALUES LESS THAN (TO_DAYS('2024-10-01')),
  PARTITION p_current VALUES LESS THAN MAXVALUE
);

-- Purge Q1 2024 instantly
ALTER TABLE events DROP PARTITION p_q1_2024;
```

No locks, no undo log growth, no replication lag. This is the most efficient purge method for high-volume tables.

## Method 3 - Schedule Purges with MySQL Events

Use the MySQL Event Scheduler to run nightly purges automatically:

```sql
-- Enable the event scheduler
SET GLOBAL event_scheduler = ON;

CREATE EVENT nightly_events_purge
ON SCHEDULE EVERY 1 DAY
STARTS '2026-04-01 02:00:00'
DO
  CALL purge_old_events();
```

Check the event is scheduled:

```sql
SHOW EVENTS FROM mydb;
```

## Monitoring Purge Impact on Replication

After each batch delete, check that replication lag is not building:

```sql
-- On the replica
SHOW REPLICA STATUS\G
```

If `Seconds_Behind_Source` grows during the purge, increase the sleep between batches:

```sql
DO SLEEP(0.2);  -- 200ms pause between batches
```

## Using pt-archiver for Purge-Only Mode

Percona's `pt-archiver` with `--purge` mode deletes rows without archiving them, providing built-in throttling:

```bash
pt-archiver \
  --source h=primary.db,D=mydb,t=events,u=root,p=secret \
  --purge \
  --where "created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)" \
  --limit 2000 \
  --sleep 0.05 \
  --check-slave-lag h=replica.db \
  --max-lag 5
```

`--check-slave-lag` pauses the purge automatically if replication lag exceeds 5 seconds.

## Summary

Purge old MySQL data with small batched deletes (2,000 rows per batch with short sleeps) to avoid locking and replication lag. For partitioned tables, use `ALTER TABLE ... DROP PARTITION` for instant zero-impact purges. Automate with MySQL Events or `pt-archiver --purge`, and monitor `Seconds_Behind_Source` on replicas to detect purge-related lag.
