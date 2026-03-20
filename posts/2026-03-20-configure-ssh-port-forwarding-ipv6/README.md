# How to Configure SSH Port Forwarding over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, SSH, Port Forwarding, Tunneling, Security

Description: Learn how to configure SSH local, remote, and dynamic port forwarding over IPv6, including binding tunnels to IPv6 addresses and forwarding to IPv6 endpoints.

## Local Port Forwarding over IPv6

Local port forwarding tunnels a local port through an SSH server to a remote destination.

```bash
# Forward local port 8080 to remote service over IPv6 SSH connection

ssh -L 8080:remote-service.local:80 user@2001:db8::10

# Bind tunnel to specific IPv6 address on local machine
ssh -L "[2001:db8::20]:8080:localhost:80" user@2001:db8::10

# Forward to an IPv6 destination through the tunnel
ssh -L "8080:[2001:db8::30]:80" user@2001:db8::10

# All IPv6: local binding, IPv6 SSH server, IPv6 destination
ssh -6 -L "[::1]:8080:[2001:db8::30]:443" user@2001:db8::10

# Multiple port forwards in one connection
ssh -L 8080:db-server:5432 \
    -L 9090:app-server:8080 \
    user@2001:db8::10
```

## Remote Port Forwarding over IPv6

Remote port forwarding exposes a local service on the remote SSH server.

```bash
# Expose local port 9090 on the remote SSH server
ssh -R 9090:localhost:9090 user@2001:db8::10

# Bind remote forward to specific IPv6 address on remote server
ssh -R "[2001:db8::10]:9090:localhost:9090" user@2001:db8::10

# GatewayPorts: Allow remote forward to be accessed by anyone
# (Requires GatewayPorts yes in remote sshd_config)
ssh -R "0.0.0.0:9090:localhost:9090" user@2001:db8::10

# IPv6 gateway on remote
ssh -R "[::]:9090:localhost:9090" user@2001:db8::10
```

## Dynamic Port Forwarding (SOCKS Proxy) over IPv6

```bash
# Create SOCKS5 proxy via IPv6 SSH server
ssh -D 1080 user@2001:db8::10

# Bind SOCKS proxy to specific IPv6 address
ssh -D "[::1]:1080" user@2001:db8::10

# Background mode (no shell)
ssh -N -D 1080 user@2001:db8::10

# Use the SOCKS proxy
curl --socks5 "[::1]:1080" http://internal-service/

# Configure browser to use SOCKS5 at ::1:1080
```

## SSH Config for Port Forwarding

```text
# ~/.ssh/config

# Database tunnel over IPv6
Host db-tunnel
    HostName 2001:db8::10
    User tunnel-user
    LocalForward 5432 db-server.internal:5432
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ExitOnForwardFailure yes

# Expose local dev server on jump host
Host dev-expose
    HostName 2001:db8::10
    User admin
    RemoteForward 8080 localhost:3000
    GatewayPorts yes

# SOCKS proxy
Host socks-proxy
    HostName 2001:db8::10
    User proxy-user
    DynamicForward 1080
    ServerAliveInterval 60
```

## sshd_config for Remote Forwarding

```text
# /etc/ssh/sshd_config (on the SSH server)

# Allow remote port forwarding to bind to all interfaces
GatewayPorts yes

# Or clientspecified - client decides the bind address
GatewayPorts clientspecified

# Allow TCP forwarding
AllowTcpForwarding yes

# Restrict specific users to forwarding only (no shell)
# Match User tunnel-user
#     ForceCommand /bin/false
#     AllowTcpForwarding yes
```

## Persistent Tunnel with AutoSSH

```bash
# Install autossh
apt install autossh  # Debian/Ubuntu

# Create persistent tunnel over IPv6
autossh -M 0 -N \
    -o "ServerAliveInterval 30" \
    -o "ServerAliveCountMax 3" \
    -L "8080:localhost:80" \
    user@2001:db8::10

# Systemd unit for persistent tunnel
# /etc/systemd/system/ssh-tunnel.service
```

```ini
[Unit]
Description=SSH IPv6 Tunnel
After=network.target

[Service]
User=tunneluser
ExecStart=/usr/bin/autossh -M 0 -N \
    -o "ServerAliveInterval 30" \
    -o "ServerAliveCountMax 3" \
    -L 8080:db-server:5432 \
    tunnel@2001:db8::10
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## Summary

SSH port forwarding works seamlessly over IPv6. Use `ssh -L "[::1]:8080:service:80" user@2001:db8::10` for local forwarding binding to an IPv6 address, and `ssh -R "[2001:db8::10]:9090:localhost:9090"` for remote forwarding on a specific IPv6 interface. For SOCKS proxies: `ssh -D "[::1]:1080" user@2001:db8::10`. Enable `GatewayPorts yes` in `sshd_config` for remote forwards accessible beyond localhost. Use `autossh` for persistent tunnels.
