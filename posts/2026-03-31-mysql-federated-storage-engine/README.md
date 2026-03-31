# What Is the FEDERATED Storage Engine in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Storage Engine, FEDERATED, Remote Table, Distributed

Description: The MySQL FEDERATED storage engine creates a local table definition that maps to a table on a remote MySQL server, enabling transparent access to remote data.

---

## Overview

The FEDERATED storage engine allows you to create a table in your local MySQL instance that is actually backed by a table on a remote MySQL server. When you query the local FEDERATED table, MySQL translates the query into a connection to the remote server and retrieves the data from there. From the application's perspective, it looks like a regular local table.

This is MySQL's built-in mechanism for accessing remote MySQL data without application-level connection management. It acts like a distributed table link.

## Enabling FEDERATED

The FEDERATED engine is disabled by default in most MySQL installations. You must enable it in the MySQL configuration and restart the server.

```ini
# In my.cnf or my.ini, under [mysqld]
federated
```

```sql
-- Verify it is enabled
SHOW ENGINES;
-- FEDERATED should show YES (not DISABLED)
```

## Creating a FEDERATED Table

First, the remote table must already exist on the remote server. Then you create a local table with the same structure pointing to it:

```sql
-- Remote server (remote-db-host) has this table:
-- CREATE TABLE products (
--   id INT NOT NULL AUTO_INCREMENT,
--   name VARCHAR(255) NOT NULL,
--   price DECIMAL(10,2) NOT NULL,
--   PRIMARY KEY (id)
-- ) ENGINE=InnoDB;

-- On the local server, create the FEDERATED table
CREATE TABLE products_remote (
  id INT NOT NULL AUTO_INCREMENT,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (id)
) ENGINE=FEDERATED
CONNECTION='mysql://remote_user:password@remote-db-host:3306/remote_db/products';
```

## Querying Remote Data

```sql
-- Reads are forwarded to the remote server
SELECT id, name, price
FROM products_remote
WHERE price < 50.00
ORDER BY price;

-- Writes are forwarded too
INSERT INTO products_remote (name, price) VALUES ('Widget', 9.99);
UPDATE products_remote SET price = 12.99 WHERE id = 101;
DELETE FROM products_remote WHERE id = 202;
```

## Using a Named Connection

You can define the connection string separately using the `CREATE SERVER` command, which avoids storing credentials in the table definition:

```sql
CREATE SERVER remote_catalog
FOREIGN DATA WRAPPER mysql
OPTIONS (
  HOST 'remote-db-host',
  DATABASE 'remote_db',
  USER 'remote_user',
  PASSWORD 'password',
  PORT 3306
);

CREATE TABLE catalog_items (
  id INT NOT NULL AUTO_INCREMENT,
  sku VARCHAR(50) NOT NULL,
  description TEXT,
  PRIMARY KEY (id)
) ENGINE=FEDERATED CONNECTION='remote_catalog/catalog_items';
```

## Limitations and Considerations

FEDERATED has important limitations to understand before using it:

- No transaction support across local and remote tables - commits on the local FEDERATED table do not coordinate with remote transactions
- No local data caching - every query hits the remote server
- No support for `ALTER TABLE` on the remote table via the FEDERATED link
- Full table scans on the local side are forwarded as full scans to the remote server - no query pushdown optimization for complex queries

```sql
-- Check if the remote connection is working
SELECT * FROM products_remote LIMIT 1;

-- FEDERATED table status shows remote connection details
SHOW TABLE STATUS LIKE 'products_remote'\G
```

## Summary

The MySQL FEDERATED storage engine enables transparent access to tables on remote MySQL servers by creating a local table definition that proxies queries over a network connection. It is useful for read-heavy cross-server data access scenarios where you want to avoid modifying application code to manage multiple database connections. However, its lack of transaction coordination, absence of local caching, and limited query optimization make it unsuitable for write-heavy or latency-sensitive workloads. For modern cross-database access patterns, consider ProxySQL or application-level data federation instead.
