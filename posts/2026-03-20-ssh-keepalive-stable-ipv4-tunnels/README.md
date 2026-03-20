# How to Configure SSH KeepAlive Settings for Stable IPv4 Tunnels

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SSH, KeepAlive, IPv4, Tunneling, Configuration, Networking, Reliability

Description: Learn how to configure SSH client and server keepalive settings to prevent idle IPv4 tunnels from being dropped by firewalls or NAT devices.

---

Idle SSH tunnels are commonly dropped by firewalls, NAT gateways, and cloud load balancers after a period of inactivity. SSH's keepalive mechanism sends periodic "null" packets to keep the connection alive and detect when the remote end has gone away.

## How SSH Keepalives Work

```
Client                    Server
  |   (idle for 30s)       |
  |--ServerAliveMessage--->|  (probe packet)
  |<-----response---------|
  |   (another 30s idle)   |
  |--ServerAliveMessage--->|
  |         (no response - server gone)
  |   (repeat CountMax times, then disconnect)
```

## Client-Side KeepAlive (ssh_config)

```bash
# ~/.ssh/config (or /etc/ssh/ssh_config for system-wide)

Host *
    # Send a keepalive packet every 30 seconds of idle time
    ServerAliveInterval 30

    # Disconnect after 3 consecutive unanswered keepalives (90 seconds total)
    ServerAliveCountMax 3

# Apply stricter settings for tunnel connections specifically
Host *-tunnel
    ServerAliveInterval 20
    ServerAliveCountMax 5
    AddressFamily inet   # Force IPv4
    TCPKeepAlive yes     # Enable OS-level TCP keepalives too
```

## Command-Line KeepAlive Options

```bash
# Start a tunnel with keepalive configured via command-line options
ssh -fN \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o TCPKeepAlive=yes \
  -L 8080:localhost:80 \
  user@203.0.113.10
```

## Server-Side KeepAlive (sshd_config)

The server can also send keepalives to detect dead clients.

```bash
# /etc/ssh/sshd_config

# Send a keepalive to the client every 60 seconds
ClientAliveInterval 60

# Disconnect after 3 unanswered keepalives (3 minutes total)
ClientAliveCountMax 3

# Enable TCP-level keepalives
TCPKeepAlive yes
```

```bash
# Validate and reload
sshd -t && systemctl reload sshd
```

## OS-Level TCP Keepalive Tuning

For very long-lived tunnels, also tune the kernel TCP keepalive parameters.

```bash
# /etc/sysctl.conf

# Start sending keepalives after 60 seconds of idle
net.ipv4.tcp_keepalive_time = 60

# Send a keepalive probe every 10 seconds
net.ipv4.tcp_keepalive_intvl = 10

# Disconnect after 6 unanswered probes (60 seconds total)
net.ipv4.tcp_keepalive_probes = 6
```

```bash
# Apply immediately
sysctl -p
```

## Recommended Settings by Use Case

| Use Case | ServerAliveInterval | ServerAliveCountMax | Notes |
|----------|-------------------|-------------------|-------|
| Corporate firewall (15m timeout) | 60s | 3 | Conservative |
| AWS/cloud (5m NAT timeout) | 60s | 5 | Start keepalives early |
| Aggressive reliability | 20s | 3 | For critical tunnels |
| Low-bandwidth links | 120s | 3 | Reduce keepalive traffic |

## Key Takeaways

- `ServerAliveInterval` is the most important setting — set it lower than your network's idle timeout.
- `ServerAliveCountMax` controls how many missed keepalives trigger a disconnect.
- `TCPKeepAlive yes` adds OS-level TCP keepalives for additional protection against dead connections.
- For cloud environments (AWS, GCP, Azure), set `ServerAliveInterval 60` to beat the typical 5-minute NAT idle timeout.
