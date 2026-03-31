# How to Configure Read-Write Splitting with MySQL Router

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MySQL Router, Read-Write Splitting, InnoDB Cluster, Performance

Description: Configure MySQL Router to automatically route write queries to the primary and read queries to replicas for horizontal read scaling in InnoDB Cluster.

---

## How MySQL Router Handles Read-Write Splitting

MySQL Router does not parse SQL to determine whether a query is a read or write. Instead, it provides two separate TCP ports:

- A **read-write port** that always connects to the cluster primary
- A **read-only port** that routes to secondaries using round-robin load balancing

Your application must direct writes to one port and reads to the other.

## Bootstrap MySQL Router

After setting up InnoDB Cluster, bootstrap MySQL Router to auto-generate configuration:

```bash
sudo mysqlrouter --bootstrap admin@node1:3306 \
  --user=mysqlrouter \
  --directory=/etc/mysqlrouter

sudo systemctl start mysqlrouter
```

## Default Port Layout

After bootstrapping, MySQL Router listens on:

```text
Port 6446 - Read/Write (classic MySQL protocol, routes to PRIMARY)
Port 6447 - Read-Only (classic MySQL protocol, routes to SECONDARY)
Port 6448 - Read/Write (X Protocol)
Port 6449 - Read-Only (X Protocol)
```

## Configure Your Application for Read-Write Splitting

```python
import mysql.connector

# Write connection - always goes to primary
write_conn = mysql.connector.connect(
    host='router-host',
    port=6446,
    user='app',
    password='secret',
    database='mydb'
)

# Read connection - distributed across replicas
read_conn = mysql.connector.connect(
    host='router-host',
    port=6447,
    user='app',
    password='secret',
    database='mydb'
)

# Use write_conn for INSERT, UPDATE, DELETE
write_cursor = write_conn.cursor()
write_cursor.execute("INSERT INTO orders (product, qty) VALUES ('Widget', 5)")
write_conn.commit()

# Use read_conn for SELECT queries
read_cursor = read_conn.cursor()
read_cursor.execute("SELECT COUNT(*) FROM products")
```

## Configure Read-Only Routing Strategy

Edit `/etc/mysqlrouter/mysqlrouter.conf` to customize how read traffic is distributed:

```ini
[routing:myCluster_ro]
bind_address = 0.0.0.0
bind_port = 6447
destinations = metadata-cache://myCluster/?role=SECONDARY
routing_strategy = round-robin-with-fallback
```

`round-robin-with-fallback` routes to secondaries but falls back to the primary if no secondaries are available.

Alternative strategies:

```ini
routing_strategy = round-robin         # Only secondaries, no fallback
routing_strategy = first-available     # Always use the first available replica
```

## Enable Client-Side Load Balancing Hint

Some ORMs support connection pool configuration for read replicas. For Django:

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'HOST': 'router-host',
        'PORT': '6446',  # writes
        ...
    },
    'replica': {
        'ENGINE': 'django.db.backends.mysql',
        'HOST': 'router-host',
        'PORT': '6447',  # reads
        ...
    }
}

DATABASE_ROUTERS = ['myapp.routers.ReadWriteRouter']
```

## Verify Connections Are Routing Correctly

Connect on each port and check which node you are connected to:

```bash
# Check write port (should show PRIMARY)
mysql -h router-host -P 6446 -u app -p \
  -e "SELECT @@hostname, @@read_only;"

# Check read port (should show SECONDARY with read_only=1)
mysql -h router-host -P 6447 -u app -p \
  -e "SELECT @@hostname, @@read_only;"
```

## Summary

MySQL Router provides read-write splitting via separate TCP ports: port 6446 for writes (primary) and port 6447 for reads (secondaries). Configure your application connection pools to use the appropriate port for each query type. Customize routing strategy in `mysqlrouter.conf` with `round-robin-with-fallback` to ensure reads fall back to the primary when no secondaries are available.
