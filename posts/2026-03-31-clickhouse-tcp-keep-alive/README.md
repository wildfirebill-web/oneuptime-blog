# How to Configure ClickHouse TCP Keep-Alive Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, TCP, Keep-Alive, Networking, Configuration

Description: Learn how to configure TCP keep-alive settings for ClickHouse to detect stale connections and prevent long-running queries from being silently dropped.

---

## What Is TCP Keep-Alive?

TCP keep-alive is an OS-level mechanism that periodically sends probe packets on idle TCP connections to detect whether the remote end is still alive. Without it, a firewall or NAT device can silently close an idle connection, causing ClickHouse clients to receive errors on the next query without any prior warning.

## Why It Matters for ClickHouse

ClickHouse native protocol (port 9000) uses long-lived TCP connections. Long-running analytical queries can hold a connection open for minutes. If a firewall has a 5-minute idle timeout and your query runs for 10 minutes, the connection will be dropped mid-query.

## Linux TCP Keep-Alive Kernel Parameters

Configure the OS-level defaults:

```bash
# Time before sending the first keep-alive probe (seconds)
sudo sysctl -w net.ipv4.tcp_keepalive_time=60

# Interval between probes (seconds)
sudo sysctl -w net.ipv4.tcp_keepalive_intvl=10

# Number of failed probes before declaring the connection dead
sudo sysctl -w net.ipv4.tcp_keepalive_probes=6
```

Make these permanent by adding to `/etc/sysctl.conf`:

```text
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
```

Apply with:

```bash
sudo sysctl -p
```

## ClickHouse Server Configuration

ClickHouse respects the OS defaults but you can also set socket options explicitly:

```xml
<tcp_port>9000</tcp_port>
```

ClickHouse does not expose direct XML settings for TCP keep-alive timing, so OS-level configuration is the primary method.

## Client-Side Settings (Python clickhouse-driver)

```python
from clickhouse_driver import Client

client = Client(
    host='clickhouse-host',
    port=9000,
    connect_timeout=10,
    send_receive_timeout=600,
    sync_request_timeout=5,
    settings={
        'tcp_keep_alive': True,
    }
)
```

## Client-Side Settings (Go)

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{"clickhouse-host:9000"},
    DialContext: func(ctx context.Context, addr string) (net.Conn, error) {
        dialer := &net.Dialer{KeepAlive: 30 * time.Second}
        return dialer.DialContext(ctx, "tcp", addr)
    },
})
```

Setting `KeepAlive` to 30 seconds means the OS sends a probe after 30 seconds of idle time.

## Verifying Keep-Alive on a Connection

```bash
# Check that keep-alive is enabled on a specific connection
ss -ti state established '( dport = :9000 or sport = :9000 )' | grep keepalive
```

## Firewall Timeout Alignment

Set your TCP keep-alive time to be significantly shorter than your firewall's idle connection timeout:

```text
Firewall idle timeout: 300 seconds (5 minutes)
Recommended tcp_keepalive_time: 60 seconds
```

This ensures keep-alive probes are sent before the firewall closes the connection.

## Summary

TCP keep-alive for ClickHouse is configured at the OS level using `net.ipv4.tcp_keepalive_*` sysctl parameters. Set `tcp_keepalive_time` to well below your firewall's idle timeout to keep long-running query connections alive. On the client side, enable `KeepAlive` in your connection dialer to benefit from OS-level probing.
