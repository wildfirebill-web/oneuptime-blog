# How to Bind SSH Tunnel to a Specific IPv4 Address Instead of localhost

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SSH, IPv4, Port Forwarding, Bind Address, GatewayPorts, Tunneling

Description: Configure SSH local port forwards to bind to a specific IPv4 address instead of the default localhost, sharing the tunnel with other machines on the network.

## Introduction

By default, SSH local port forwards (`-L`) bind to `127.0.0.1`, making the tunnel accessible only from the local machine. Binding to a network IPv4 address shares the tunnel with other clients on the same network.

## Default vs. Specific Bind Address

```bash
# Default: bind to 127.0.0.1 only (accessible only locally)

ssh -L 5432:db.internal:5432 user@203.0.113.10

# Bind to specific IPv4 (accessible from 10.0.0.5 on the network)
ssh -L 10.0.0.5:5432:db.internal:5432 user@203.0.113.10

# Bind to all interfaces (accessible from any machine that can reach you)
ssh -L 0.0.0.0:5432:db.internal:5432 user@203.0.113.10
```

## Enabling Non-Localhost Bind

This requires `GatewayPorts` or sshd configuration:

```bash
# /etc/ssh/sshd_config (on the SSH server, NOT the local machine)
# Note: For LOCAL forwards, GatewayPorts controls the LOCAL bind on the CLIENT side
# This is controlled by the client's sshd config when using -R

# For local forwards from client to server, the client controls the bind:
# No server-side config change needed for -L
```

For local port forwards (`-L`), the client simply specifies the bind address:

```bash
# Share database tunnel with entire 10.0.0.0/8 subnet
ssh -4 -fN \
  -L 10.0.0.5:5432:db.internal:5432 \
  user@203.0.113.10

# Other machines can now:
# psql -h 10.0.0.5 -p 5432 -U dbuser mydb
```

## Using AllowStreamLocalForwarding in sshd

```bash
# sshd_config on the SSH server
AllowTcpForwarding yes
AllowStreamLocalForwarding yes
GatewayPorts clientspecified   # For -R with specific bind address
```

## SSH Config with Specific Bind

```bash
# ~/.ssh/config

Host shared-db-tunnel
    HostName 203.0.113.10
    User admin
    AddressFamily inet

    # Bind on network interface, not loopback - shares with team
    LocalForward 10.0.0.5:5432 db.internal:5432
    LocalForward 10.0.0.5:6379 redis.internal:6379

    ServerAliveInterval 30
    ServerAliveCountMax 3
```

## Firewall Rules for Tunnel Port

When binding to a network IP, add firewall rules:

```bash
# Allow team members to access the tunnel port
sudo iptables -A INPUT -s 10.0.0.0/24 -p tcp --dport 5432 -j ACCEPT

# Block everyone else from the tunnel port
sudo iptables -A INPUT -p tcp --dport 5432 -j DROP
```

## Testing the Specific Bind

```bash
# Verify tunnel binds to network IP
ss -tlnp | grep :5432
# Expected: LISTEN 0 128 10.0.0.5:5432 0.0.0.0:*

# Test from a different machine on the network
psql -h 10.0.0.5 -p 5432 -U dbuser mydb

# Test connectivity
nc -zv 10.0.0.5 5432
```

## Conclusion

Binding SSH local port forwards to a specific IPv4 address (e.g., `10.0.0.5:5432`) instead of `127.0.0.1:5432` shares the tunnel with other machines on your network. Specify the bind address as `<local-IP>:<local-port>:<remote-host>:<remote-port>` in the `-L` flag. Combine with firewall rules to restrict which source IPs can use the shared tunnel.
