# How to Fix "Connection refused" Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Networking, Configuration, Connection

Description: Diagnose and fix "Connection refused" errors when connecting to ClickHouse by checking service status, port bindings, firewall rules, and listen address configuration.

---

"Connection refused" means the TCP connection to ClickHouse was actively rejected. This is almost always caused by ClickHouse not running, listening on the wrong interface, or a firewall blocking the connection.

## Step 1: Check if ClickHouse is Running

```bash
sudo systemctl status clickhouse-server
```

If it is not running:

```bash
sudo systemctl start clickhouse-server
sudo journalctl -u clickhouse-server -n 50
```

Look for startup errors in the journal that prevented the service from starting.

## Step 2: Verify Port Binding

Check what ports ClickHouse is actually listening on:

```bash
sudo ss -tlnp | grep clickhouse
# or
sudo netstat -tlnp | grep clickhouse
```

Expected output:

```text
LISTEN  0  128  127.0.0.1:8123  0.0.0.0:*  users:(("clickhouse-s",pid=...,fd=...))
LISTEN  0  128  127.0.0.1:9000  0.0.0.0:*  users:(("clickhouse-s",pid=...,fd=...))
```

If ClickHouse is only listening on `127.0.0.1` and you are connecting from a remote host, you need to change the `listen_host`.

## Step 3: Fix the listen_host Configuration

Edit or create `/etc/clickhouse-server/config.d/listen.xml`:

```xml
<clickhouse>
  <listen_host>0.0.0.0</listen_host>
  <!-- Or bind to a specific interface -->
  <!-- <listen_host>192.168.1.10</listen_host> -->
</clickhouse>
```

Restart ClickHouse:

```bash
sudo systemctl restart clickhouse-server
```

## Step 4: Check Firewall Rules

```bash
# iptables
sudo iptables -L -n | grep -E "8123|9000"

# ufw
sudo ufw status

# firewalld
sudo firewall-cmd --list-all
```

Allow the ports:

```bash
# ufw
sudo ufw allow from 10.0.0.0/8 to any port 8123
sudo ufw allow from 10.0.0.0/8 to any port 9000

# firewalld
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" port protocol="tcp" port="8123" accept'
sudo firewall-cmd --reload
```

## Step 5: Check Cloud Security Groups

If running on a cloud provider, the security group or firewall rule may block the connection. Verify that inbound rules allow TCP on port 8123 or 9000 from your client IP.

## Step 6: Test Connectivity

From the server itself:

```bash
clickhouse-client --host 127.0.0.1 --port 9000 --query "SELECT 1"
curl 'http://127.0.0.1:8123/?query=SELECT+1'
```

From the client host:

```bash
nc -zv clickhouse-host 8123
nc -zv clickhouse-host 9000
telnet clickhouse-host 8123
```

## Step 7: Correct Port in Client

Verify you are using the right port for your client:

```text
HTTP interface:     8123
HTTPS interface:    8443
Native TCP:         9000
Native TCP + TLS:   9440
MySQL protocol:     9004
PostgreSQL protocol: 9005
```

## Summary

"Connection refused" errors when connecting to ClickHouse are resolved by verifying the service is running, checking that ClickHouse is bound to `0.0.0.0` or the correct interface in `listen_host`, ensuring firewall rules permit the connection, and verifying the client is using the correct port for the protocol (8123 for HTTP, 9000 for native TCP).
