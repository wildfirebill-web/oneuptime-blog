# How to Tune Redis for Low-Latency Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Performance, Latency, Tuning, Configuration

Description: Learn how to configure Redis and the underlying OS to achieve sub-millisecond latency for latency-sensitive workloads like session stores and real-time systems.

---

Redis is fast by default, but achieving consistent sub-millisecond latency requires tuning both Redis configuration and the OS. Tail latency (P99, P99.9) is often more impactful than average latency.

## Disable Persistence (If Tolerable)

RDB saves and AOF fsync are the biggest sources of latency spikes:

```text
# /etc/redis/redis.conf

# Disable RDB snapshots
save ""

# Disable AOF
appendonly no
```

If you need durability, use `appendfsync everysec` instead of `always`.

## Disable Transparent Huge Pages

THP causes `fork()` latency spikes during RDB saves:

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

Make it persistent in `/etc/rc.local` or a systemd unit.

## Set CPU Affinity

Pin Redis to dedicated CPU cores to reduce scheduling jitter:

```bash
# Pin Redis to cores 2 and 3
taskset -cp 2,3 $(pgrep redis-server)
```

Or configure in `redis.conf`:

```text
server-cpulist 0-3
bio-cpulist 4-5
```

## Tune the Event Loop

Redis 6+ supports multi-threaded I/O:

```text
# /etc/redis/redis.conf
io-threads 4
io-threads-do-reads yes
```

Enable only if CPU is the bottleneck on your specific workload.

## Optimize TCP Settings

```text
# /etc/redis/redis.conf
tcp-backlog 511
tcp-keepalive 60
```

At the OS level:

```bash
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

## Use Unix Domain Sockets for Local Clients

Unix sockets bypass the TCP stack:

```text
# /etc/redis/redis.conf
unixsocket /var/run/redis/redis.sock
unixsocketperm 770
```

```bash
redis-cli -s /var/run/redis/redis.sock PING
```

## Disable Slow Log Sampling

In very latency-sensitive cases, even slow log overhead matters:

```text
slowlog-log-slower-than -1
```

Set to a high value like `10000` (10ms) in production to only log genuinely slow commands.

## Monitor Latency

```bash
redis-cli --latency -h 127.0.0.1 -p 6379
redis-cli --latency-history -h 127.0.0.1 -p 6379 -i 5
```

## Summary

To minimize Redis latency: disable or defer persistence, turn off Transparent Huge Pages, pin to dedicated CPU cores, enable TCP tuning, and use Unix sockets for local connections. The biggest single improvement for low-latency workloads is eliminating RDB fork spikes and THP-related stalls.
