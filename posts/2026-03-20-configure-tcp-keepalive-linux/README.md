# How to Configure TCP Keepalive Settings on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, Keepalive, Networking, Connection, Kernel

Description: Configure TCP keepalive parameters on Linux to detect dead connections and prevent long-lived connections from being terminated by firewalls and load balancers.

## Introduction

TCP keepalive sends small probe packets on idle connections to verify that both endpoints are still alive. Without keepalives, an idle TCP connection can be silently killed by a firewall's stateful timeout or a NAT device, causing the application to see a "broken pipe" or "connection reset" error minutes later when it tries to use the connection.

## How TCP Keepalive Works

After a connection has been idle for `tcp_keepalive_time` seconds, the kernel sends keepalive probes every `tcp_keepalive_intvl` seconds, up to `tcp_keepalive_probes` times. If no response is received, the connection is considered dead and closed.

## Kernel-Level Keepalive Configuration

```bash
# View current keepalive settings

sysctl net.ipv4.tcp_keepalive_time    # Idle time before starting probes (default: 7200s = 2hr)
sysctl net.ipv4.tcp_keepalive_intvl   # Interval between probes (default: 75s)
sysctl net.ipv4.tcp_keepalive_probes  # Number of probes before declaring dead (default: 9)

# Default: 2 hours idle + 9 probes × 75s = ~2 hours 11 minutes to detect dead connection

# Tune for faster dead connection detection
sysctl -w net.ipv4.tcp_keepalive_time=60      # Start probing after 60s idle
sysctl -w net.ipv4.tcp_keepalive_intvl=10     # Probe every 10 seconds
sysctl -w net.ipv4.tcp_keepalive_probes=6     # 6 probes before declaring dead
# Result: detects dead connection in 60 + (6 × 10) = ~120 seconds

# Persist settings
cat >> /etc/sysctl.conf << EOF
net.ipv4.tcp_keepalive_time=60
net.ipv4.tcp_keepalive_intvl=10
net.ipv4.tcp_keepalive_probes=6
EOF
sysctl -p
```

## Enabling Keepalive per Socket in Applications

Kernel keepalive settings only apply to sockets with `SO_KEEPALIVE` enabled. Enable it at the application level:

```python
import socket

def create_socket_with_keepalive():
    """Create a TCP socket with keepalive enabled."""
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # Enable keepalive on this socket
    s.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)

    # Override kernel defaults for this socket (Linux-specific)
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 60)   # Start keepalive after 60 seconds idle
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)  # Probe every 10 seconds
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 6)     # 6 probes before giving up

    return s
```

## Enabling Keepalive in Databases and Servers

```bash
# PostgreSQL: enable keepalive for client connections
# postgresql.conf:
tcp_keepalives_idle = 60
tcp_keepalives_interval = 10
tcp_keepalives_count = 6

# nginx: enable keepalive for upstream connections
upstream backend {
    server 10.20.0.5:8080;
    keepalive 32;    # Keep up to 32 connections to backend alive
}

# Redis: configure via redis.conf
# tcp-keepalive 60
```

## Verifying Keepalive is Active

```bash
# Check that SO_KEEPALIVE is set on a socket
ss -tnop | grep "keepalive"

# Monitor keepalive probes being sent
tcpdump -i eth0 -n 'tcp and not (tcp[tcpflags] & (tcp-push|tcp-fin) != 0)'
# Keepalive probes appear as ACK packets with seq = last-seq - 1

# Check for connections using keepalive
ss -tnop | grep timer:keepalive
```

## Conclusion

TCP keepalives are essential for long-lived connections in environments with firewalls, NAT devices, or load balancers that have their own idle timeouts. The key trade-off is between keepalive frequency (more probes = faster detection, more traffic) and resource efficiency. For database connections and persistent API clients, set keepalive idle to slightly less than the firewall's timeout (typically 30-60 seconds).
