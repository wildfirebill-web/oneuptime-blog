# How to Use libcephsqlite for SQLite on RADOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, SQLite, libcephsqlite, RADOS, Database, Storage

Description: Use libcephsqlite to store SQLite databases directly in Ceph RADOS, enabling replicated, distributed SQLite storage without a shared filesystem.

---

`libcephsqlite` is a SQLite VFS (Virtual File System) extension that stores SQLite database files as RADOS objects in Ceph. This allows you to use SQLite with Ceph's replication, durability, and distributed access - without needing a mounted filesystem.

## What libcephsqlite Provides

- SQLite databases stored as RADOS objects (no filesystem mount needed)
- Replication and durability from Ceph (no separate backup for the SQLite file)
- Concurrent access coordination using RADOS locking primitives
- Standard SQLite API - no application code changes required beyond the VFS registration

## Prerequisites

```bash
# libcephsqlite is included with Ceph (Quincy+)
# Verify the library is installed
ls /usr/lib/x86_64-linux-gnu/libcephsqlite.so

# Or install it
apt install libcephsqlite  # Debian/Ubuntu
dnf install libcephsqlite  # RHEL/CentOS
```

## Loading libcephsqlite in SQLite

```bash
# From the SQLite CLI
sqlite3

# Load the extension
.load /usr/lib/x86_64-linux-gnu/libcephsqlite

# Open a database in Ceph
# Format: file:<pool>/<object-name>?vfs=ceph
.open file:mypool/myapp.db?vfs=ceph

# Use it like any SQLite database
CREATE TABLE events (id INTEGER PRIMARY KEY, ts TEXT, message TEXT);
INSERT INTO events (ts, message) VALUES (datetime('now'), 'test entry');
SELECT * FROM events;
```

## Using libcephsqlite from Python

```python
import sqlite3
import ctypes

# Load the libcephsqlite extension
libcephsqlite = ctypes.CDLL("/usr/lib/x86_64-linux-gnu/libcephsqlite.so")

# Connect using the ceph VFS
conn = sqlite3.connect("file:mypool/myapp.db?vfs=ceph", uri=True)

conn.execute("""
    CREATE TABLE IF NOT EXISTS metrics (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        host TEXT NOT NULL,
        metric TEXT NOT NULL,
        value REAL NOT NULL,
        recorded_at TEXT DEFAULT (datetime('now'))
    )
""")

conn.execute("INSERT INTO metrics (host, metric, value) VALUES (?, ?, ?)",
             ("web-01", "cpu_pct", 42.5))
conn.commit()

rows = conn.execute("SELECT * FROM metrics ORDER BY recorded_at DESC LIMIT 10").fetchall()
for row in rows:
    print(row)

conn.close()
```

## Verifying the RADOS Objects

The SQLite database is stored as RADOS objects in the pool:

```bash
# List objects in the pool
rados -p mypool ls | grep myapp

# You should see objects like:
# myapp.db
# myapp.db-wal  (if WAL mode is enabled)
# myapp.db-shm  (shared memory file)
```

## Ceph Configuration for libcephsqlite

By default, libcephsqlite reads `/etc/ceph/ceph.conf`. You can override this:

```bash
# Set via environment variable
export CEPH_CONF=/path/to/custom/ceph.conf
export CEPH_KEYRING=/path/to/keyring

# Or use a URI parameter
sqlite3 "file:mypool/myapp.db?vfs=ceph&ceph_config=/etc/ceph/ceph.conf"
```

## Summary

`libcephsqlite` makes it possible to use SQLite as an application database backed by Ceph RADOS, gaining Ceph's replication and durability without requiring a mounted filesystem. The library implements the SQLite VFS interface, so existing SQLite code works without modification - just load the extension and use the `ceph` VFS in the connection URI. Database files are stored as RADOS objects in a Ceph pool, making them accessible from any node that can reach the cluster.
