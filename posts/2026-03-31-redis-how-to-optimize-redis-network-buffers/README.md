# How to Optimize Redis Network Buffers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Performance, Networking, Tuning, Linux

Description: Learn how to tune Redis network buffer settings and Linux TCP parameters to improve throughput and reduce latency in high-traffic deployments.

---

Redis network buffer tuning affects how much data can be queued for reading and writing per client connection. Proper buffer configuration prevents dropped connections under high load and improves throughput. This guide covers both Redis-level and OS-level buffer settings.

## Redis Client Output Buffer Limits

The `client-output-buffer-limit` directive controls how much data Redis buffers per client before disconnecting them.

```text
# redis.conf
# Format: client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

- `normal` - regular clients (0 means no limit)
- `replica` - replica sync buffers (256mb hard, 64mb soft with 60s grace)
- `pubsub` - pub/sub subscribers (32mb hard, 8mb soft with 60s grace)

Increase these if replicas or subscribers are frequently disconnecting:

```bash
redis-cli CONFIG SET client-output-buffer-limit "replica 512mb 128mb 60"
```

## TCP Backlog

The `tcp-backlog` setting controls how many pending TCP connections Redis will queue before refusing new ones:

```text
# redis.conf
tcp-backlog 511
```

For high-concurrency deployments, increase this value:

```text
tcp-backlog 4096
```

Also raise the OS kernel limit to match:

```bash
echo 4096 | sudo tee /proc/sys/net/core/somaxconn
echo 4096 | sudo tee /proc/sys/net/ipv4/tcp_max_syn_backlog
```

To persist across reboots, add to `/etc/sysctl.conf`:

```text
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
```

## TCP Keep-Alive

Enable TCP keep-alive to detect dead connections and free up resources:

```text
# redis.conf
tcp-keepalive 300
```

This sends keep-alive probes every 300 seconds. A value of 0 disables it.

## Linux TCP Socket Buffers

Increase OS-level TCP socket buffer sizes for high-throughput Redis usage:

```bash
# View current settings
sysctl net.core.rmem_max
sysctl net.core.wmem_max
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem
```

Tune them for Redis workloads:

```bash
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.wmem_max=16777216
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
```

Add to `/etc/sysctl.conf` to persist:

```text
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

Apply without reboot:

```bash
sudo sysctl -p
```

## Monitoring Buffer Utilization

Check client output buffer usage in real time:

```bash
redis-cli CLIENT LIST | grep -i omem
```

The `omem` field shows the current output buffer size per client. Large values indicate a slow consumer.

Use `INFO clients` for aggregate stats:

```bash
redis-cli INFO clients
# Key fields:
# blocked_clients
# tracking_clients
# client_recent_max_output_buffer
# client_recent_max_input_buffer
```

## Disabling Nagle's Algorithm

Disable Nagle's algorithm to reduce latency for small packets:

```text
# redis.conf
tcp-nodelay yes
```

This is the default in Redis. Do not disable it unless you are on a slow WAN link and want to reduce packet count.

## Summary

Optimize Redis network buffers by adjusting `client-output-buffer-limit` for replicas and pub/sub clients, increasing `tcp-backlog` alongside the OS `somaxconn` limit, and tuning Linux TCP socket buffer sizes for high-throughput workloads. Monitor `omem` in `CLIENT LIST` and `INFO clients` to detect slow consumers before they cause disconnections.
