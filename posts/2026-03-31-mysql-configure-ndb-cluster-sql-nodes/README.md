# How to Configure MySQL NDB Cluster SQL Nodes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, NDB Cluster, SQL Node, Configuration, mysqld

Description: Learn how to configure MySQL SQL nodes in NDB Cluster to enable the ndbcluster storage engine and connect them to the cluster management node.

---

## Role of SQL Nodes

SQL nodes in NDB Cluster are standard MySQL server instances (`mysqld`) with the NDB storage engine plugin loaded. They provide the SQL interface for applications connecting to the cluster. Multiple SQL nodes can run simultaneously, giving horizontal read/write scalability. SQL nodes do not store data themselves - they forward operations to data nodes.

## SQL Node my.cnf Configuration

On each SQL node host, configure `/etc/mysql/my.cnf`:

```text
[mysqld]
# Standard MySQL settings
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
user=mysql
port=3306

# NDB Cluster settings
ndbcluster
ndb-connectstring=192.168.1.10

[mysql_cluster]
ndb-connectstring=192.168.1.10
```

The critical settings are:
- `ndbcluster` - enables the NDB storage engine
- `ndb-connectstring` - points to the management node IP or hostname

## Management Node config.ini Entry for SQL Nodes

Each SQL node must have a matching `[mysqld]` entry in the management node's `config.ini`:

```text
[mysqld]
NodeId=4
hostname=192.168.1.13

[mysqld]
NodeId=5
hostname=192.168.1.14
```

## Starting the SQL Node

```bash
sudo systemctl start mysql
```

Or:

```bash
mysqld_safe --user=mysql &
```

## Verifying NDB Engine is Loaded

After connecting to the SQL node:

```sql
SHOW ENGINES;
```

Look for:

```text
Engine  | Support | Comment
NDBCLUSTER | YES | Clustered, fault-tolerant tables
```

## Creating an NDB Table

```sql
CREATE DATABASE clusterdb;
USE clusterdb;

CREATE TABLE orders (
    id         INT NOT NULL AUTO_INCREMENT,
    customer   VARCHAR(100) NOT NULL,
    amount     DECIMAL(10,2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id)
) ENGINE=NDBCLUSTER;
```

## Verifying Cluster Connectivity

```sql
SHOW STATUS LIKE 'ndb_connected';
```

```text
Variable_name  | Value
ndb_connected  | ON
```

Check the node ID:

```sql
SHOW STATUS LIKE 'ndb_cluster_node_id';
```

## SQL Node-Specific Tuning

Add these to `my.cnf` for better SQL node performance:

```text
[mysqld]
ndbcluster
ndb-connectstring=192.168.1.10

# Increase API batch sizes
ndb-batch-size=32768

# Allow more concurrent NDB transactions
ndb-cluster-connection-pool=4

# Connection pool for multi-threaded access
ndb-cluster-connection-pool-nodeids=4,5
```

## Checking Connected SQL Nodes

From the management node:

```bash
ndb_mgm -e show
```

Connected SQL nodes show:

```text
[mysqld(API)]   2 node(s)
id=4    @192.168.1.13  (mysql-8.0.36 ndb-8.0.36)
id=5    @192.168.1.14  (mysql-8.0.36 ndb-8.0.36)
```

## Summary

SQL nodes are standard MySQL servers with the NDB engine enabled via `ndbcluster` in `my.cnf` and a connection string pointing to the management node. After starting, verify the NDB engine shows as `YES` in `SHOW ENGINES` and `ndb_connected` is `ON`. Create tables with `ENGINE=NDBCLUSTER` to store them in the distributed cluster rather than on local disk.
