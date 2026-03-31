# How to Fix "Connection refused" Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Connection, Error, Troubleshooting, Network

Description: Diagnose and fix "Connection refused" errors when connecting to ClickHouse, covering server state, binding, firewall, and TLS issues.

---

A "Connection refused" error means the TCP connection to ClickHouse's port was actively rejected by the host. This is different from a timeout - the server responded, but nothing is listening on the requested port.

## Verify the Server is Running

```bash
sudo systemctl status clickhouse-server
# or
ps aux | grep clickhouse-server
```

If the server is stopped, start it:

```bash
sudo systemctl start clickhouse-server
```

Check for startup errors:

```bash
sudo tail -100 /var/log/clickhouse-server/clickhouse-server.err.log
```

## Check Which Ports ClickHouse is Listening On

```bash
sudo ss -tlnp | grep clickhouse
# Default ports: 9000 (native), 8123 (HTTP), 9440 (native TLS), 8443 (HTTPS)
```

If nothing appears, ClickHouse failed to bind. Review the config:

```bash
grep -E "listen_host|tcp_port|http_port" /etc/clickhouse-server/config.xml
```

## Fix Binding to All Interfaces

By default ClickHouse only listens on localhost. To accept remote connections:

```xml
<!-- /etc/clickhouse-server/config.xml -->
<listen_host>0.0.0.0</listen_host>
```

Restart after changes:

```bash
sudo systemctl restart clickhouse-server
```

## Check Firewall Rules

```bash
# iptables
sudo iptables -L INPUT -n | grep -E "9000|8123"

# UFW
sudo ufw status
sudo ufw allow 9000/tcp
sudo ufw allow 8123/tcp

# firewalld
sudo firewall-cmd --list-ports
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --reload
```

## Test Connectivity

```bash
# Test native protocol
nc -zv 127.0.0.1 9000

# Test HTTP interface
curl -v http://127.0.0.1:8123/ping
```

## Port Conflict Resolution

Another process may be using the same port:

```bash
sudo lsof -i :9000
sudo lsof -i :8123
```

If there is a conflict, change ClickHouse's port in `config.xml`:

```xml
<tcp_port>9001</tcp_port>
```

## Summary

"Connection refused" to ClickHouse means the server is not listening on the expected port. Start by confirming the server is running and checking its logs. Ensure ClickHouse is configured to listen on `0.0.0.0` for remote access, verify firewall rules allow the required ports, and confirm no other process is occupying them.
