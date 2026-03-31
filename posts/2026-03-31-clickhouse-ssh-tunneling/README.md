# How to Set Up ClickHouse with SSH Tunneling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SSH, Tunneling, Security, Networking

Description: Learn how to access ClickHouse securely over SSH tunnels, enabling encrypted connections without exposing ports or configuring a VPN.

---

## Why SSH Tunneling for ClickHouse?

SSH tunneling provides an encrypted channel to reach ClickHouse on servers that are not publicly exposed. It is ideal for developers who need temporary secure access to a production or staging ClickHouse instance without setting up a VPN or modifying firewall rules.

## Basic SSH Port Forwarding

Forward your local port 8123 to the remote ClickHouse HTTP port:

```bash
ssh -L 8123:localhost:8123 user@clickhouse-server
```

While the SSH session is open, queries sent to `localhost:8123` are forwarded to ClickHouse on the remote server:

```bash
curl "http://localhost:8123/?query=SELECT+1"
clickhouse-client --host localhost --port 8123 --query "SELECT hostName()"
```

## Forwarding the Native TCP Port

For the native ClickHouse protocol (port 9000):

```bash
ssh -L 9000:localhost:9000 user@clickhouse-server
```

Then connect with `clickhouse-client` on default settings (connects to `localhost:9000`):

```bash
clickhouse-client --query "SELECT count() FROM system.tables"
```

## Background Tunnel

Run the tunnel in the background without an interactive shell:

```bash
ssh -f -N -L 8123:localhost:8123 user@clickhouse-server
```

- `-f`: Fork into background after authentication
- `-N`: Do not execute a remote command (tunnel only)

Kill the tunnel:

```bash
# Find the PID
ps aux | grep "ssh -f"
kill <pid>
```

## Persistent Tunnel with autossh

For a tunnel that automatically reconnects after network interruptions:

```bash
sudo apt install autossh

autossh -M 20000 -f -N \
  -L 8123:localhost:8123 \
  -o "ServerAliveInterval=30" \
  -o "ServerAliveCountMax=3" \
  user@clickhouse-server
```

## SSH Config File for Convenience

Add a shortcut to `~/.ssh/config`:

```text
Host ch-tunnel
    HostName clickhouse-server.example.com
    User ubuntu
    IdentityFile ~/.ssh/clickhouse_key
    LocalForward 8123 localhost:8123
    LocalForward 9000 localhost:9000
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

Then connect with:

```bash
ssh -N ch-tunnel
```

## Tunneling Through a Bastion Host

If ClickHouse is behind a bastion (jump host):

```bash
ssh -L 8123:ch-internal:8123 -J bastion-user@bastion-host user@ch-internal
```

Or with ProxyJump in `~/.ssh/config`:

```text
Host ch-internal
    HostName 10.0.0.5
    User ubuntu
    ProxyJump bastion-user@bastion.example.com
    LocalForward 8123 localhost:8123
```

## Summary

SSH tunneling is a quick and secure way to access ClickHouse without exposing ports or configuring a VPN. Use `ssh -L` for temporary access, `autossh` for persistent tunnels, and the `ProxyJump` option to traverse bastion hosts. For production environments, consider a VPN for always-on access and reserve SSH tunneling for developer and admin access.
