# How to Fix "Connection refused" Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Connection Refused, Troubleshooting, Networking, Configuration

Description: Fix ClickHouse "Connection refused" errors by diagnosing server status, port configuration, firewall rules, and network interface binding issues.

---

## Overview

"Connection refused" errors when connecting to ClickHouse typically indicate that the ClickHouse server is not running, is not listening on the expected port, or a firewall is blocking the connection. The default ClickHouse ports are `9000` (native protocol), `8123` (HTTP interface), and `9440` (native TLS).

## Check If ClickHouse Is Running

```bash
# Check service status
sudo systemctl status clickhouse-server

# Check if process is running
ps aux | grep clickhouse-server

# Check which ports ClickHouse is listening on
sudo ss -tlnp | grep clickhouse
# or
sudo netstat -tlnp | grep 9000
```

## Start ClickHouse If Not Running

```bash
sudo systemctl start clickhouse-server
sudo systemctl enable clickhouse-server

# Check server logs for startup errors
sudo journalctl -u clickhouse-server -f
sudo tail -100 /var/log/clickhouse-server/clickhouse-server.err.log
```

## Verify Port Configuration

```xml
<!-- /etc/clickhouse-server/config.xml - default ports -->
<clickhouse>
    <tcp_port>9000</tcp_port>
    <http_port>8123</http_port>
    <interserver_http_port>9009</interserver_http_port>
</clickhouse>
```

Check if a custom config overrides these:

```bash
grep -r "tcp_port\|http_port" /etc/clickhouse-server/config.d/
```

## Check Network Interface Binding

By default, ClickHouse 20.6+ listens only on `localhost` (`127.0.0.1`). For remote connections, configure it to listen on all interfaces:

```xml
<!-- /etc/clickhouse-server/config.d/listen.xml -->
<clickhouse>
    <listen_host>0.0.0.0</listen_host>
    <!-- or specific interface -->
    <listen_host>10.0.1.5</listen_host>
</clickhouse>
```

Restart after changing:

```bash
sudo systemctl restart clickhouse-server
```

## Check Firewall Rules

```bash
# UFW (Ubuntu)
sudo ufw status
sudo ufw allow 9000/tcp
sudo ufw allow 8123/tcp

# firewalld (RHEL/CentOS)
sudo firewall-cmd --list-ports
sudo firewall-cmd --add-port=9000/tcp --permanent
sudo firewall-cmd --add-port=8123/tcp --permanent
sudo firewall-cmd --reload

# iptables
sudo iptables -L -n | grep 9000
sudo iptables -A INPUT -p tcp --dport 9000 -j ACCEPT
```

## Test Connectivity

```bash
# Test HTTP interface
curl -v http://localhost:8123/

# Test native protocol port
nc -zv localhost 9000

# Test from remote host
curl -v http://clickhouse-host:8123/
nc -zv clickhouse-host 9000
```

## Check User Network Restrictions

ClickHouse users can have IP-based network restrictions in `users.xml`:

```xml
<users>
    <default>
        <networks>
            <ip>::1</ip>
            <ip>127.0.0.1</ip>
            <!-- Add your application server's IP -->
            <ip>10.0.0.0/8</ip>
        </networks>
    </default>
</users>
```

If the connecting IP is not in this list, the connection will be refused.

## Docker/Kubernetes Networking Issues

In Docker, ensure the port is exposed:

```bash
docker run -d \
    -p 9000:9000 \
    -p 8123:8123 \
    clickhouse/clickhouse-server
```

In Kubernetes, check the Service configuration:

```bash
kubectl get svc clickhouse -n clickhouse
kubectl describe svc clickhouse -n clickhouse
```

## Check ClickHouse Server Logs

```bash
# View recent server errors
sudo tail -100 /var/log/clickhouse-server/clickhouse-server.err.log

# Check for startup errors
sudo grep -i "error\|exception\|fatal" /var/log/clickhouse-server/clickhouse-server.log | tail -50
```

## Summary

ClickHouse "Connection refused" errors are caused by the server not running, listening only on localhost while a remote connection is attempted, firewall rules blocking the port, or user network restrictions in `users.xml`. Diagnose by checking `systemctl status clickhouse-server`, verifying port binding with `ss -tlnp`, and testing connectivity with `curl` (HTTP) or `nc` (TCP). Add `<listen_host>0.0.0.0</listen_host>` to enable remote access and update firewall rules to allow ports 9000 and 8123.
