# How to Tune TCP Keepalive Parameters on Linux for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP Keepalive, Linux, sysctl, IPv4, Networking, Connection Tuning, Performance

Description: Learn how to tune TCP keepalive parameters on Linux to detect dead connections, prevent NAT table timeouts, and maintain long-lived IPv4 connections in production environments.

---

TCP keepalive probes detect dead connections without application-layer heartbeats. Tuning these parameters prevents NAT expiry, firewall timeouts, and resource leaks from zombie connections.

## Default TCP Keepalive Parameters

```bash
# View current values
sysctl net.ipv4.tcp_keepalive_time
sysctl net.ipv4.tcp_keepalive_intvl
sysctl net.ipv4.tcp_keepalive_probes

# Defaults:
# tcp_keepalive_time   = 7200  (2 hours before first probe)
# tcp_keepalive_intvl  = 75    (75s between probes)
# tcp_keepalive_probes = 9     (9 probes before giving up)
```

The default 2-hour idle time is too long for most environments with NAT or firewalls.

## Tuning for Production (Short-Lived NAT/Firewall Tables)

```bash
# Reduce keepalive time to 60 seconds
sysctl -w net.ipv4.tcp_keepalive_time=60

# Probe every 10 seconds
sysctl -w net.ipv4.tcp_keepalive_intvl=10

# Give up after 5 failed probes (total: 60 + 5*10 = 110s before DROP)
sysctl -w net.ipv4.tcp_keepalive_probes=5
```

## Making Changes Persistent

```bash
# /etc/sysctl.d/99-tcp-keepalive.conf
net.ipv4.tcp_keepalive_time   = 60
net.ipv4.tcp_keepalive_intvl  = 10
net.ipv4.tcp_keepalive_probes = 5

# Apply
sysctl -p /etc/sysctl.d/99-tcp-keepalive.conf
```

## Enabling TCP Keepalive on Application Sockets

The kernel only sends keepalive probes on sockets where `SO_KEEPALIVE` is set. Most server software has a configuration option:

**Nginx:**
```nginx
http {
    # Keepalive to upstream backends
    keepalive_timeout 65;  # HTTP keepalive, not TCP keepalive
}
# TCP keepalive is controlled by the OS socket options
```

**PostgreSQL (client connection):**
```ini
# postgresql.conf
tcp_keepalives_idle     = 60
tcp_keepalives_interval = 10
tcp_keepalives_count    = 5
```

**Redis:**
```ini
# redis.conf
tcp-keepalive 60
```

## Verifying Keepalive on Active Connections

```bash
# Check if SO_KEEPALIVE is set on a TCP socket
ss -tnop | grep ESTABLISHED | grep keepalive

# Example output:
# ESTAB 0 0 10.0.0.1:443 10.0.0.50:54321 users:(("nginx",pid=1234,fd=12)) timer:(keepalive,58sec,0)
```

## Diagnosing Keepalive Issues

```bash
# Capture keepalive probes with tcpdump
tcpdump -i eth0 -nn "tcp and port 443 and (tcp[tcpflags] & tcp-ack != 0) and (ip[2:2] < 60)"

# Check for RST after keepalive failure
tcpdump -i eth0 -nn "tcp[tcpflags] & tcp-rst != 0"
```

## Key Takeaways

- Reduce `tcp_keepalive_time` from the default 2 hours to 60 seconds in environments with NAT or stateful firewalls.
- Set `SO_KEEPALIVE` at the application layer or use application-level keepalive settings for PostgreSQL, Redis, and similar services.
- Use `ss -tnop` to verify keepalive timers are active on established connections.
- Formula for dead connection detection time: `tcp_keepalive_time + (tcp_keepalive_probes × tcp_keepalive_intvl)`.
