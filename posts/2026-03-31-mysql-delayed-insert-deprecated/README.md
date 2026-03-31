# How to Use DELAYED INSERT in MySQL (Deprecated)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DELAYED INSERT, MyISAM, Deprecation

Description: Learn what DELAYED INSERT was, how it worked in MySQL for MyISAM tables, why it was deprecated in MySQL 5.6 and removed in MySQL 5.7, and the modern alternatives.

---

## What Was DELAYED INSERT

`INSERT DELAYED` was a MySQL extension that allowed an `INSERT` statement to return immediately to the client while MySQL queued the actual row insertion to be performed in the background. The goal was to reduce client latency for non-critical writes such as logging or analytics events on **MyISAM** tables.

```sql
-- MySQL 5.5 and earlier: return immediately, insert later
INSERT DELAYED INTO myisam_event_log (event_type, message, created_at)
VALUES ('PAGE_VIEW', '/home', NOW());
```

The server queued this row and inserted it when the table was not otherwise in use.

## Why It Was Removed

- **MyISAM-only** - `INSERT DELAYED` never worked with InnoDB, NDB, or other storage engines
- **No guarantee of execution** - if the server crashed before flushing the queue, queued rows were lost
- **No LAST_INSERT_ID** - the function returned 0 because the row was not yet inserted
- **Deprecated in MySQL 5.6.6**, removed entirely in **MySQL 5.7**

In MySQL 5.7+, using `INSERT DELAYED` causes a warning that the modifier is ignored, and the insert is executed synchronously.

## Behavior in MySQL 5.7 and 8.0

```sql
-- MySQL 5.7/8.0: DELAYED is silently ignored, insert runs synchronously
INSERT DELAYED INTO log_table (message) VALUES ('test');
-- Warning: 1287 'INSERT DELAYED' is deprecated and will be removed in a future release.
-- Please use INSERT (without DELAYED).
```

## Modern Alternatives

### 1. InnoDB with Asynchronous Application-Side Queuing

The most common replacement is to insert into an InnoDB table normally from the application and use an application-level queue (like Redis or a message broker) for deferred processing:

```sql
-- Normal synchronous InnoDB insert
INSERT INTO event_log (event_type, message, created_at)
VALUES ('PAGE_VIEW', '/home', NOW());
```

### 2. Batch Inserts

Collect multiple rows in the application and insert them in a single statement to reduce per-row overhead:

```sql
-- Insert multiple rows in one statement - efficient and synchronous
INSERT INTO event_log (event_type, message, created_at) VALUES
  ('PAGE_VIEW',   '/home',    NOW()),
  ('PAGE_VIEW',   '/about',   NOW()),
  ('BUTTON_CLICK','signup',   NOW());
```

### 3. INSERT with LOW_PRIORITY on MyISAM

If you are still using MyISAM, `LOW_PRIORITY` defers writes until no reads are pending, which reduces read interference without losing rows:

```sql
INSERT LOW_PRIORITY INTO myisam_log (message) VALUES ('background event');
```

### 4. Application-Level Write Buffering in Python

```python
import queue
import threading
import pymysql

write_queue = queue.Queue()

def flush_worker():
    conn = pymysql.connect(host="127.0.0.1", user="app", password="pass", database="mydb")
    while True:
        batch = []
        try:
            while len(batch) < 100:
                row = write_queue.get(timeout=0.1)
                batch.append(row)
        except queue.Empty:
            pass
        if batch:
            with conn.cursor() as cur:
                cur.executemany(
                    "INSERT INTO event_log (event_type, message) VALUES (%s, %s)",
                    batch
                )
            conn.commit()

threading.Thread(target=flush_worker, daemon=True).start()

# Application writes go to the queue, not the database directly
write_queue.put(("PAGE_VIEW", "/home"))
```

## Summary

`INSERT DELAYED` was a MyISAM-specific optimization for non-blocking background inserts, deprecated in MySQL 5.6 and removed in MySQL 5.7. Modern applications should use InnoDB with standard inserts, batch inserts for high-throughput logging, or application-side write queues backed by a message broker. These alternatives are more reliable, work with all storage engines, and provide proper durability guarantees.
