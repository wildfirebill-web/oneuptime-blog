# What Is MySQL and How Does It Work

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, SQL

Description: Learn what MySQL is, how its client-server architecture works, what the InnoDB storage engine does, and why MySQL is the world's most popular open-source database.

---

MySQL is an open-source relational database management system (RDBMS) that stores data in tables with rows and columns. It uses SQL (Structured Query Language) to query and manipulate data. MySQL is the world's most widely deployed open-source database, powering applications from small websites to global platforms.

## Client-Server Architecture

MySQL operates as a server process that listens for connections on a TCP port (default: 3306) or a Unix socket. Client applications connect, send SQL statements, and receive results.

```bash
# Start MySQL (systemd)
sudo systemctl start mysql

# Connect using the MySQL CLI client
mysql -h 127.0.0.1 -u root -p

# Check server status
mysqladmin -u root -p status
```

The MySQL server handles authentication, authorization, query parsing, optimization, and execution. Clients can be command-line tools, ORMs, or any driver that implements the MySQL protocol.

## How a Query Executes

When you run a SQL statement, MySQL processes it through several layers:

1. **Connection handler** - authenticates the client and allocates a thread
2. **Parser** - tokenizes and parses the SQL into an internal syntax tree
3. **Optimizer** - chooses the best execution plan (which indexes to use, join order)
4. **Storage engine** - reads/writes actual data

```sql
-- This SELECT goes through all layers
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id
ORDER BY order_count DESC
LIMIT 10;
```

## Storage Engines

MySQL separates the SQL layer from the storage layer via a pluggable storage engine API. Different engines handle data storage differently.

```sql
-- Check available storage engines
SHOW ENGINES;

-- InnoDB is the default and recommended engine
CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10,2)
) ENGINE=InnoDB;
```

**InnoDB** is the default engine. It provides ACID transactions, row-level locking, foreign key constraints, and crash recovery. Most tables should use InnoDB.

## Data Storage on Disk

InnoDB stores data in tablespace files. By default, each table gets its own `.ibd` file when `innodb_file_per_table` is enabled.

```bash
# InnoDB data files
ls /var/lib/mysql/myapp/
# products.ibd
# users.ibd
# orders.ibd
```

The InnoDB buffer pool caches frequently accessed pages in RAM, reducing disk I/O.

```sql
-- Check buffer pool usage
SHOW STATUS LIKE 'Innodb_buffer_pool%';
-- Innodb_buffer_pool_pages_data: pages currently in use
-- Innodb_buffer_pool_read_requests: total read requests
-- Innodb_buffer_pool_reads: reads that required disk access
```

## Binary Log

MySQL records every change to data in the binary log (`binlog`). This enables:
- **Replication** - replicas read the binlog to apply changes
- **Point-in-time recovery** - replay binlog events after a backup to recover to a specific moment

```sql
-- View binary log files
SHOW BINARY LOGS;

-- Enable binlog (in my.cnf)
-- log_bin = /var/log/mysql/mysql-bin.log
-- server_id = 1
```

## Basic SQL Operations

```sql
-- Create a database and table
CREATE DATABASE shop;
USE shop;
CREATE TABLE products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  price DECIMAL(10,2)
);

-- Insert, query, update, delete
INSERT INTO products (name, price) VALUES ('Widget', 9.99);
SELECT * FROM products WHERE price < 20;
UPDATE products SET price = 12.99 WHERE name = 'Widget';
DELETE FROM products WHERE id = 1;
```

## Summary

MySQL is a client-server relational database that parses SQL, optimizes query execution, and uses pluggable storage engines to persist data. InnoDB is the default engine and provides full ACID compliance, row-level locking, and crash recovery. The binary log enables replication and point-in-time recovery. MySQL's combination of performance, reliability, and open-source availability makes it the default choice for web applications worldwide.
