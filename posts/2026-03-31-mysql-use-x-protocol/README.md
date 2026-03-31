# How to Use MySQL X Protocol

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, X Protocol, Network, Plugin, Connection

Description: Learn how the MySQL X Protocol works, how to configure it, and how to connect using both the X DevAPI and raw protocol clients on port 33060.

---

## What is the MySQL X Protocol?

The MySQL X Protocol is a modern, binary network protocol introduced in MySQL 5.7.12 that operates on port 33060 by default. It powers the X DevAPI and supports full-duplex, asynchronous communication between clients and the MySQL server. Unlike the classic MySQL protocol (port 3306), the X Protocol is built on Protocol Buffers (protobuf) and supports:

- CRUD operations on document collections
- CRUD operations on SQL tables
- Asynchronous query execution
- Row streaming
- Authentication via SHA-256 and cached SHA-2 plugins

## Verifying the X Plugin is Active

```sql
SHOW PLUGINS WHERE Name = 'mysqlx';
```

Expected output:

```text
Name    | Status | Type       | Library     | License
mysqlx  | ACTIVE | DAEMON     | mysqlx.so   | GPL
```

## Checking the X Protocol Port

```sql
SHOW VARIABLES LIKE 'mysqlx_port';
```

```text
Variable_name | Value
mysqlx_port   | 33060
```

## Enabling or Installing the Plugin Manually

If the plugin is not active:

```sql
INSTALL PLUGIN mysqlx SONAME 'mysqlx.so';
```

Or add to `/etc/mysql/my.cnf` and restart:

```text
[mysqld]
mysqlx=ON
mysqlx-port=33060
mysqlx-socket=/var/run/mysqld/mysqlx.sock
```

## Connecting via MySQL Shell

```bash
# Using the X Protocol (mysqlx://)
mysqlsh --uri mysqlx://root@127.0.0.1:33060

# Classic protocol for comparison
mysqlsh --uri mysql://root@127.0.0.1:3306
```

## Connecting via Node.js X DevAPI

```javascript
const mysqlx = require('@mysql/xdevapi');

const session = await mysqlx.getSession('mysqlx://root:secret@127.0.0.1:33060/mydb');
console.log('Connected via X Protocol');
await session.close();
```

## Connecting via Python X DevAPI

```python
import mysqlx

session = mysqlx.get_session({
    "host": "127.0.0.1",
    "port": 33060,
    "user": "root",
    "password": "secret"
})
print("Connected:", session.is_open())
session.close()
```

## Configuring the Bind Address and Socket

```text
[mysqld]
mysqlx-bind-address=0.0.0.0
mysqlx-port=33060
mysqlx-socket=/tmp/mysqlx.sock
```

To restrict X Protocol to localhost only:

```text
[mysqld]
mysqlx-bind-address=127.0.0.1
```

## Monitoring X Protocol Connections

```sql
SHOW STATUS LIKE 'Mysqlx%';
```

Key metrics:

```text
Mysqlx_connections_accepted
Mysqlx_connections_closed
Mysqlx_connections_rejected
Mysqlx_crud_insert
Mysqlx_crud_find
Mysqlx_crud_update
Mysqlx_crud_delete
```

## Disabling the X Protocol

If the X Protocol is not needed:

```text
[mysqld]
mysqlx=OFF
```

Or:

```sql
UNINSTALL PLUGIN mysqlx;
```

## Summary

The MySQL X Protocol provides a modern, efficient communication layer for the X DevAPI on port 33060. Enable it by default on MySQL 8.0, verify with `SHOW PLUGINS`, and connect using `mysqlx://` URIs from MySQL Shell or supported language connectors. Monitor connection metrics via `SHOW STATUS LIKE 'Mysqlx%'` and restrict the bind address to localhost if remote X Protocol access is not needed.
