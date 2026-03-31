# How to Fix 'Network is unreachable' for ClickHouse Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Network, Replication, Error, Troubleshooting

Description: Fix 'Network is unreachable' errors in ClickHouse replication by diagnosing routing, DNS, firewall rules, and inter-server port access.

---

"Network is unreachable" errors in ClickHouse replication indicate that the server cannot establish a TCP connection to a replica. Unlike "Connection refused" (where the port is closed), this error means the network path itself is broken.

## Identify the Unreachable Host

Check the replication queue for errors:

```sql
SELECT
    source_replica,
    last_exception,
    type,
    create_time
FROM system.replication_queue
WHERE last_exception != ''
ORDER BY create_time DESC
LIMIT 10;
```

Also check system.replicas:

```sql
SELECT
    database,
    table,
    host_name,
    host_port,
    is_currently_executing,
    last_exception
FROM system.replicas
WHERE last_exception != '';
```

## Test Network Connectivity

From the ClickHouse server:

```bash
# Test ICMP (ping) - may be blocked by firewalls
ping -c 3 clickhouse-replica2

# Test TCP port connectivity (more reliable)
nc -zv clickhouse-replica2 9009  # Inter-server HTTP port
nc -zv clickhouse-replica2 9000  # Native protocol
```

## Verify Routing

```bash
ip route get 10.0.1.20  # Replace with replica IP
traceroute 10.0.1.20
```

If there is no route, add one:

```bash
sudo ip route add 10.0.1.0/24 via 10.0.0.1 dev eth0
```

## Check Inter-Server HTTP Port

ClickHouse replication uses port 9009 (inter-server HTTP) in addition to 9000. Both must be accessible:

```xml
<!-- config.xml -->
<interserver_http_port>9009</interserver_http_port>
<interserver_http_host>clickhouse-replica1.internal</interserver_http_host>
```

The `interserver_http_host` must be a resolvable hostname or IP that other replicas can reach.

## Fix Firewall Rules

```bash
# Allow replication ports between cluster nodes
sudo ufw allow from 10.0.0.0/8 to any port 9000
sudo ufw allow from 10.0.0.0/8 to any port 9009

# Or for iptables:
sudo iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 9009 -j ACCEPT
sudo iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 9000 -j ACCEPT
```

## Fix Multi-NIC Binding

If the server has multiple network interfaces, ensure ClickHouse advertises the correct IP:

```xml
<interserver_http_host>10.0.1.10</interserver_http_host>
```

Use the internal network IP, not `localhost` or `127.0.0.1`.

## Summary

"Network is unreachable" in ClickHouse replication points to a broken network path. Verify routing tables, test both port 9000 and 9009 connectivity between replicas, configure `interserver_http_host` to the correct network interface IP, and ensure firewall rules permit inter-cluster traffic. Use `system.replication_queue` to identify which replicas are failing.
