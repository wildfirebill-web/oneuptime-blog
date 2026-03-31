# How to Use the MySQL Memcached Plugin

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Memcached, Plugin

Description: Learn how to install and configure the MySQL InnoDB Memcached plugin to access InnoDB tables directly via the Memcached protocol for ultra-low-latency reads.

---

The MySQL InnoDB Memcached plugin allows applications to access InnoDB table data directly using the Memcached protocol, bypassing the SQL parsing layer entirely. This provides much lower latency for simple key-value lookups compared to standard SQL queries, while keeping data stored in InnoDB with full ACID properties.

## How It Works

The plugin exposes a Memcached-compatible interface (port 11211 by default) that maps Memcached `get`/`set`/`delete` operations to InnoDB table reads and writes. Data is stored in an InnoDB table but accessible via either SQL or the Memcached protocol.

## Prerequisites

The plugin requires MySQL 5.6+ with InnoDB. Enable the daemon_memcached plugin:

```bash
# Load the schema required by the plugin
mysql -u root -p < /usr/share/mysql/innodb_memcached_config.sql
```

## Installing the Plugin

```sql
-- Install the plugin
INSTALL PLUGIN daemon_memcached SONAME 'libmemcached.so';

-- Verify installation
SELECT plugin_name, plugin_status FROM information_schema.plugins
WHERE plugin_name = 'daemon_memcached';
```

## Configuring the Mapping

The `innodb_memcache.containers` table maps Memcached key-value operations to InnoDB columns:

```sql
USE innodb_memcache;

INSERT INTO containers (
  name, db_schema, db_table, key_columns, value_columns,
  flags, cas_column, expire_time_column, unique_idx_name_on_key
) VALUES (
  'default',      -- container name (use 'default' for the default mapping)
  'myapp',        -- database name
  'kv_store',     -- table name
  'id',           -- key column
  'value',        -- value column(s), comma-separated for multi-column
  'flags',        -- flags column
  'cas_col',      -- CAS column (for compare-and-swap)
  'expire_time',  -- expiration column
  'PRIMARY'       -- index name
);
```

## Creating the Backing InnoDB Table

```sql
USE myapp;

CREATE TABLE kv_store (
  id          VARCHAR(255) NOT NULL,
  value       TEXT,
  flags       INT UNSIGNED NOT NULL DEFAULT 0,
  cas_col     BIGINT UNSIGNED NOT NULL DEFAULT 0,
  expire_time INT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (id)
) ENGINE=InnoDB;
```

## Configuring the Plugin in my.cnf

```ini
[mysqld]
loose-daemon_memcached_option="-p11211"
loose-daemon_memcached_r_batch_size=1
loose-daemon_memcached_w_batch_size=1
```

Restart MySQL after making these changes.

## Accessing Data via Memcached Protocol

```bash
# Set a value
printf "set mykey 0 0 5\r\nhello\r\n" | nc 127.0.0.1 11211

# Get a value
printf "get mykey\r\n" | nc 127.0.0.1 11211
```

Using a Memcached client in Python:

```python
import pymemcache.client.base as memcache

# Connect to the MySQL Memcached plugin port
mc = memcache.Client(('127.0.0.1', 11211))

# Write directly to InnoDB via Memcached protocol
mc.set('session:abc123', 'user_id=42&role=admin', expire=3600)

# Read back
value = mc.get('session:abc123')
print(value)  # b'user_id=42&role=admin'
```

The data is immediately visible via SQL as well:

```sql
SELECT id, value FROM myapp.kv_store WHERE id = 'session:abc123';
```

## Multi-Column Mapping

To store structured data across multiple columns, list them in `value_columns`:

```sql
UPDATE innodb_memcache.containers
SET value_columns = 'first_name,last_name,email'
WHERE name = 'users';
```

Access as a pipe-delimited value: `Alice|Smith|alice@example.com`.

## Performance Considerations

The Memcached plugin is best suited for simple key-value lookups and session storage. Complex queries, joins, and aggregations still require the SQL interface. Benchmark both paths to determine if the latency reduction justifies the added operational complexity.

## Summary

The MySQL InnoDB Memcached plugin provides a Memcached-compatible interface to InnoDB tables, enabling lower-latency key-value access while maintaining SQL visibility and ACID guarantees. It is most useful for session storage and hot key-value lookups where SQL parsing overhead is a measurable bottleneck.
