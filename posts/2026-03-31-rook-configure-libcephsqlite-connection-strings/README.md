# How to Configure libcephsqlite Connection Strings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, SQLite, libcephsqlite, Connection String, Configuration, RADOS

Description: Master libcephsqlite connection string syntax and options to configure pool, Ceph config path, auth credentials, and VFS parameters for SQLite on RADOS.

---

`libcephsqlite` uses SQLite's URI filename format to specify where the database lives in Ceph and how to authenticate. Understanding the connection string syntax is essential for correctly configuring database placement, auth, and behavior.

## URI Format Overview

The libcephsqlite connection string follows SQLite's URI format:

```yaml
file:<pool>/<object-name>?vfs=ceph[&<option>=<value>...]
```

Key components:
- `<pool>` - the RADOS pool where the database object is stored
- `<object-name>` - the RADOS object name (the SQLite database file)
- `vfs=ceph` - required to activate the libcephsqlite VFS

## Basic Connection String

```bash
# Minimal connection - pool and object name
sqlite3 "file:mypool/app.db?vfs=ceph"

# With WAL mode for better write concurrency
sqlite3 "file:mypool/app.db?vfs=ceph" <<EOF
PRAGMA journal_mode=WAL;
SELECT * FROM sqlite_master;
EOF
```

## Specifying a Custom Ceph Config

```bash
# Point to a non-default ceph.conf
sqlite3 "file:mypool/app.db?vfs=ceph&ceph_config=/path/to/ceph.conf"

# Use a specific keyring file
sqlite3 "file:mypool/app.db?vfs=ceph&ceph_keyring=/etc/ceph/ceph.client.myapp.keyring"
```

## Setting the Ceph Auth User

```bash
# Use a specific Ceph user (client.myapp -> id=myapp)
sqlite3 "file:mypool/app.db?vfs=ceph&ceph_user=myapp"
```

## Python Examples with Different Connection Strings

```python
import sqlite3

# Default configuration (uses /etc/ceph/ceph.conf and client.admin)
conn = sqlite3.connect("file:data/myapp.db?vfs=ceph", uri=True)

# Custom user and config
conn = sqlite3.connect(
    "file:app-pool/metrics.db?vfs=ceph"
    "&ceph_user=myapp"
    "&ceph_config=/etc/ceph/ceph.conf",
    uri=True
)

# Read-only connection (use ?mode=ro)
conn = sqlite3.connect(
    "file:data/config.db?vfs=ceph&mode=ro",
    uri=True
)
```

## Environment Variable Configuration

You can set Ceph connection parameters via environment variables that libcephsqlite reads:

```bash
export CEPH_CONF=/etc/ceph/ceph.conf
export CEPH_KEYRING=/etc/ceph/ceph.client.myapp.keyring
export CEPH_USER=myapp

# Now connection strings can be simpler
sqlite3 "file:data/app.db?vfs=ceph"
```

## Creating the Auth User and Pool

Before using a connection string, ensure the pool and user exist:

```bash
# Create the pool
ceph osd pool create mypool 32
ceph osd pool application enable mypool cephsqlite

# Create an auth user for the app
ceph auth get-or-create client.myapp \
  mon 'allow r' \
  osd 'allow rwx pool=mypool' \
  -o /etc/ceph/ceph.client.myapp.keyring
```

## Ceph Namespace Support

Use a RADOS namespace within a pool to organize databases:

```bash
# The namespace is part of the object path in libcephsqlite
# Format: file:<pool>/<namespace>/<object>?vfs=ceph
sqlite3 "file:mypool/app1/config.db?vfs=ceph"
sqlite3 "file:mypool/app2/config.db?vfs=ceph"

# Verify objects in the namespace
rados -p mypool -N app1 ls
rados -p mypool -N app2 ls
```

## Connection String Validation

```python
import sqlite3

def validate_ceph_connection(pool: str, db_name: str, user: str = "admin") -> bool:
    uri = f"file:{pool}/{db_name}?vfs=ceph&ceph_user={user}"
    try:
        conn = sqlite3.connect(uri, uri=True, timeout=5)
        conn.execute("SELECT 1")
        conn.close()
        return True
    except sqlite3.OperationalError as e:
        print(f"Connection failed: {e}")
        return False

validate_ceph_connection("mypool", "test.db", "myapp")
```

## Summary

libcephsqlite connection strings use SQLite's URI format with `?vfs=ceph` to route the connection through the RADOS backend. The pool and object name are embedded in the path, while optional parameters control which Ceph config, keyring, and user to use. Setting auth via environment variables simplifies connection strings in application code. Always create the pool and Ceph auth user with appropriate OSD caps before attempting connections, and test connectivity with a simple `SELECT 1` before deploying to production.
