# How to Troubleshoot Redis Connection Refused Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Troubleshooting, Networking, Connection, DevOps

Description: Diagnose and fix Redis "Connection refused" errors by checking if Redis is running, bind address, firewall rules, and client configuration.

---

## Understanding Connection Refused

A "Connection refused" error means the TCP connection attempt was actively rejected by the target host. In Redis contexts this normally means one of: Redis is not running, Redis is listening on a different port or interface, a firewall is blocking the port, or the client is pointing at the wrong host or port.

This differs from a timeout (host is reachable but not responding) or a DNS failure.

## Step 1 - Verify Redis is Running

```bash
# Check process status
ps aux | grep redis-server

# With systemd
systemctl status redis

# With Docker
docker ps | grep redis
```

If Redis is not running, start it:

```bash
systemctl start redis
# or
redis-server /etc/redis/redis.conf
```

## Step 2 - Check the Listening Port and Interface

```bash
# Show what address and port Redis is bound to
ss -tlnp | grep redis
# or
netstat -tlnp | grep redis
```

Expected output for a default installation:

```text
LISTEN  0  511  127.0.0.1:6379  0.0.0.0:*  users:(("redis-server",pid=1234,fd=6))
```

If you see `127.0.0.1:6379` but your client is connecting from a different host, Redis is only accepting local connections. Check your `redis.conf`:

```bash
grep -E '^bind' /etc/redis/redis.conf
```

To allow remote connections, change the bind directive and restart:

```text
bind 0.0.0.0
```

## Step 3 - Check Firewall Rules

```bash
# iptables
iptables -L INPUT -n | grep 6379

# ufw
ufw status | grep 6379

# firewalld
firewall-cmd --list-ports | grep 6379
```

Allow the port if needed:

```bash
# ufw
ufw allow 6379/tcp

# firewalld
firewall-cmd --permanent --add-port=6379/tcp
firewall-cmd --reload
```

## Step 4 - Test Connectivity from the Client Host

```bash
# Basic TCP connectivity test
nc -zv <redis-host> 6379

# Using redis-cli directly
redis-cli -h <redis-host> -p 6379 PING
```

If `nc` fails but the host is pingable, the issue is the firewall or Redis bind address. If `nc` succeeds but `redis-cli` fails, the issue is authentication or TLS configuration.

## Step 5 - Check for Port Conflicts

```bash
ss -tlnp | grep 6379
```

If a different process is holding port 6379, Redis will fail to start or start on a different port. Identify the conflicting process and resolve the conflict before starting Redis.

## Step 6 - Check Redis Logs

```bash
# Default log location
tail -100 /var/log/redis/redis-server.log

# journald
journalctl -u redis -n 100
```

Common log messages that indicate the root cause:

```text
# Port already in use
Creating Server TCP listening socket 127.0.0.1:6379: bind: Address already in use

# Permission denied
Creating Server TCP listening socket 127.0.0.1:6379: bind: Permission denied
```

## Step 7 - Docker and Kubernetes Specific Checks

In Docker, verify the port is published:

```bash
docker inspect <container> | grep -A5 Ports
```

In Kubernetes, verify the Service and Pod are healthy:

```bash
kubectl get svc redis
kubectl get pods -l app=redis
kubectl describe pod <redis-pod>
```

Check if the Service selector matches Pod labels:

```bash
kubectl get endpoints redis
```

An empty Endpoints list means the selector does not match any running Pods.

## Common Fixes Summary

| Symptom | Fix |
|---|---|
| Redis not running | `systemctl start redis` |
| Bound to 127.0.0.1, remote client | Set `bind 0.0.0.0` in redis.conf |
| Firewall blocking port | Open port 6379 in firewall |
| Wrong host/port in client | Correct client configuration |
| Port conflict | Resolve conflicting process |
| Docker port not published | Add `-p 6379:6379` to docker run |

## Summary

Redis "Connection refused" errors are almost always caused by Redis not running, a misconfigured bind address, or a firewall rule. Start by confirming the Redis process is alive and listening on the expected interface, then test TCP connectivity from the client host before investigating firewall rules and client configuration. Checking Redis logs usually reveals the root cause in under a minute.
