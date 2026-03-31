# How to Configure MySQL Router for Load Balancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MySQL Router, Load Balancing, Read/Write Splitting

Description: Learn how to configure MySQL Router's routing strategies to distribute read queries across multiple replicas for load balancing.

---

## Load Balancing with MySQL Router

MySQL Router supports several routing strategies that control how connections are distributed across backend servers. This is particularly useful for spreading read-only workloads across multiple replicas.

## Routing Strategies

| Strategy | Description |
|---|---|
| `first-available` | Connects to the first server in the list; tries next if unavailable |
| `next-available` | Like first-available, but never goes back to a previous server |
| `round-robin` | Distributes connections evenly across all available servers |
| `round-robin-with-fallback` | Round-robin across secondaries; falls back to primary if all secondaries are down |

## Configuring Read/Write Splitting

The standard load-balancing setup routes writes to the primary and distributes reads across replicas:

```ini
# /etc/mysqlrouter/mysqlrouter.conf
[DEFAULT]
logging_folder = /var/log/mysqlrouter

[routing:rw]
bind_address     = 0.0.0.0
bind_port        = 6446
destinations     = primary.db.internal:3306
routing_strategy = first-available
protocol         = classic

[routing:ro]
bind_address     = 0.0.0.0
bind_port        = 6447
destinations     = replica1.db.internal:3306,replica2.db.internal:3306,replica3.db.internal:3306
routing_strategy = round-robin
protocol         = classic
```

## Using Metadata Cache for Dynamic Discovery

When using InnoDB Cluster, use metadata-cache destinations to auto-discover servers:

```ini
[metadata_cache:myCluster]
router_id     = 1
bootstrap_server_addresses = node1:3306,node2:3306,node3:3306
user          = clusteradmin
metadata_cluster = myCluster
ttl           = 5

[routing:rw]
bind_address     = 0.0.0.0
bind_port        = 6446
destinations     = metadata-cache://myCluster/?role=PRIMARY
routing_strategy = first-available
protocol         = classic

[routing:ro]
bind_address     = 0.0.0.0
bind_port        = 6447
destinations     = metadata-cache://myCluster/?role=SECONDARY
routing_strategy = round-robin-with-fallback
protocol         = classic
```

`ttl = 5` means Router refreshes cluster topology every 5 seconds.

## Application-Side Load Balancing

Your application must direct query types to the correct port:

```python
import mysql.connector

# Write connection (to primary via port 6446)
write_conn = mysql.connector.connect(
    host='router.internal', port=6446,
    user='appuser', password='secret', database='mydb'
)

# Read connection (to replicas via port 6447, round-robin)
read_conn = mysql.connector.connect(
    host='router.internal', port=6447,
    user='appuser', password='secret', database='mydb'
)

# Use write_conn for INSERT/UPDATE/DELETE
# Use read_conn for SELECT
```

## Connection Pooling Configuration

MySQL Router also supports connection pooling to reduce overhead:

```ini
[connection_pool]
max_idle_server_connections = 64
```

This keeps idle server connections alive, reducing handshake overhead for new client requests.

## Monitoring Router Load Distribution

Check Router logs to see connection routing:

```bash
tail -f /var/log/mysqlrouter/mysqlrouter.log | grep "routing"
```

To see current routing destinations and their health:

```bash
curl http://localhost:8081/api/20190715/routes/ro/status
```

Enable the REST API in `mysqlrouter.conf`:

```ini
[http_server]
port = 8081
bind_address = 127.0.0.1

[rest_api]

[rest_routing]
require_realm = default_auth
```

## Summary

MySQL Router distributes read load across replicas using the `round-robin` or `round-robin-with-fallback` strategy on the read-only port (6447), while directing writes to the primary on port 6446. Using metadata-cache with InnoDB Cluster enables dynamic server discovery so Router automatically adjusts when the topology changes.
