# How to Troubleshoot Redis Replication Disconnections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Replication, Troubleshooting, Disconnection, Monitoring

Description: Learn how to diagnose and fix Redis replication disconnections by analyzing logs, checking buffers, network health, and configuration parameters.

---

Redis replication disconnections cause replicas to fall behind or trigger expensive full resyncs. This guide walks through diagnosing why disconnections happen and how to fix them.

## Step 1 - Check Replication State

Start with `INFO replication` on both primary and replica:

```bash
redis-cli -h primary INFO replication
redis-cli -h replica INFO replication
```

On the primary, look for:

```text
connected_slaves:0          # replica is disconnected
slave0: state=wait_bgsave   # replica is reconnecting
```

On the replica:

```text
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:1   # full resync underway
```

## Step 2 - Check Redis Logs

```bash
tail -f /var/log/redis/redis-server.log
```

Common messages and their causes:

```text
# Output buffer overflow
Client id=... addr=... cmd=psync scheduled to be closed ASAP for overcoming of output buffer limits

# Network timeout
Connection with replica ... lost

# Replica initiated disconnect
Replica ... asks for synchronization
```

## Step 3 - Output Buffer Configuration

A common cause of disconnections is the output buffer on the primary filling up when the replica is slow:

```bash
redis-cli CONFIG GET client-output-buffer-limit
```

Default for replicas:

```text
replica 256mb 64mb 60
# 256mb hard limit, or 64mb sustained for 60 seconds -> disconnect
```

Increase these limits for fast-writing primaries:

```bash
redis-cli CONFIG SET client-output-buffer-limit "replica 512mb 128mb 120"
```

## Step 4 - Check TCP Buffer Sizes

Replication uses persistent TCP connections. Small kernel buffers cause issues on high-latency networks:

```bash
# Check current settings
sysctl net.core.rmem_max net.core.wmem_max

# Increase if needed
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.wmem_max=16777216
```

In `/etc/sysctl.conf` for persistence:

```text
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

## Step 5 - Replication Timeout

If the primary is performing a slow operation (large BGSAVE, heavy Lua script) the replica may time out:

```bash
redis-cli CONFIG GET repl-timeout
# default: 60 seconds
```

Increase for large datasets or slow networks:

```bash
redis-cli CONFIG SET repl-timeout 120
```

## Step 6 - Backlog Too Small

If disconnections are brief but triggering full resyncs, the backlog is too small:

```bash
redis-cli INFO replication | grep backlog
redis-cli INFO stats | grep sync_full
```

If `sync_full` is increasing frequently, increase the backlog:

```bash
redis-cli CONFIG SET repl-backlog-size 134217728  # 128MB
```

## Monitoring for Disconnections

Set up alerts on `master_link_status:down` using a monitoring loop:

```bash
while true; do
  STATUS=$(redis-cli -h replica INFO replication | grep master_link_status)
  echo "$(date): $STATUS"
  sleep 5
done
```

## Summary

Redis replication disconnections are typically caused by output buffer limits, TCP buffer constraints, replication timeouts, or an undersized backlog. Diagnose by checking logs and `INFO replication`, then tune `client-output-buffer-limit`, `repl-timeout`, and `repl-backlog-size` to match your workload characteristics.
