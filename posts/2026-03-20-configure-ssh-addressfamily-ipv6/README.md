# How to Configure SSH AddressFamily for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, SSH, OpenSSH, AddressFamily, Configuration

Description: Learn how to configure the SSH AddressFamily directive in sshd_config and ssh_config to control whether SSH uses IPv4, IPv6, or both for connections.

## AddressFamily Directive

The `AddressFamily` directive controls which address families SSH will use. It applies to both the client (`~/.ssh/config` or `/etc/ssh/ssh_config`) and server (`/etc/ssh/sshd_config`).

```
# Values:
# any   - Use both IPv4 and IPv6 (default)
# inet  - IPv4 only
# inet6 - IPv6 only
```

## sshd_config (Server Configuration)

```
# /etc/ssh/sshd_config

# IPv6 only server
AddressFamily inet6
ListenAddress ::

# Or bind to specific IPv6 address
AddressFamily inet6
ListenAddress 2001:db8::10

# Dual-stack server (default)
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

# IPv4 only (legacy)
AddressFamily inet
ListenAddress 0.0.0.0
```

## Client ssh_config (System-wide)

```
# /etc/ssh/ssh_config

# Default IPv6 for all users
Host *
    AddressFamily inet6

# Or per-host settings
Host production-server
    HostName prod.example.com
    AddressFamily inet6
    User deploy
```

## User ~/.ssh/config (Per-User)

```
# ~/.ssh/config

# Global: prefer IPv6
Host *
    AddressFamily inet6

# Specific host - IPv6 only
Host ipv6-server
    HostName 2001:db8::10
    User admin
    AddressFamily inet6
    IdentityFile ~/.ssh/id_ed25519

# Specific host - dual stack
Host dual-stack-server
    HostName server.example.com
    User admin
    AddressFamily any  # Try IPv6 first, fall back to IPv4

# Override global setting for legacy host
Host legacy-server
    HostName old.example.com
    AddressFamily inet  # IPv4 only for this host
```

## Applying sshd Changes

```bash
# Test sshd_config syntax
sshd -t

# View current effective configuration
sshd -T | grep -i addressfamily

# Reload sshd (no connection drops)
systemctl reload sshd

# Verify after reload
ss -tlnp | grep sshd
ss -6 -tlnp | grep sshd  # IPv6 listeners
ss -4 -tlnp | grep sshd  # IPv4 listeners
```

## Effects on Connection Behavior

```bash
# With AddressFamily inet6 in ssh_config:
# This will ONLY try AAAA records, IPv4 is ignored
ssh user@server.example.com

# With AddressFamily any (default):
# DNS returns both A and AAAA → order depends on system policy
# Typically tries IPv6 first on modern systems
ssh user@dual-stack.example.com

# Verify which address family was used
ssh -v user@server.example.com 2>&1 | grep "Connecting to"
```

## Troubleshooting AddressFamily Issues

```bash
# Problem: "connect: Network is unreachable" with inet6
# Cause: No IPv6 connectivity
# Fix: Set AddressFamily any or inet

# Check if IPv6 is available
ip -6 route show default
ping6 -c 1 2001:4860:4860::8888  # Google IPv6 DNS

# Problem: SSH tries IPv6 but times out before trying IPv4
# Fix: Add ConnectTimeout and set AddressFamily any
# ~/.ssh/config
# Host *
#     AddressFamily any
#     ConnectTimeout 10

# Problem: sshd not listening on IPv6 after setting inet6
# Check ListenAddress is set to IPv6 address or ::
sshd -T | grep -E "(addressfamily|listenaddress)"

# Verify sshd listening
ss -6 -tlnp | grep ':22'
```

## Per-Command AddressFamily Override

```bash
# Override client AddressFamily per-command
ssh -o "AddressFamily inet6" user@server.example.com
ssh -o "AddressFamily inet" user@server.example.com
ssh -o "AddressFamily any" user@server.example.com

# Equivalent short flags
ssh -6 user@server.example.com   # inet6
ssh -4 user@server.example.com   # inet
```

## Summary

The `AddressFamily` directive in SSH controls address selection: `inet6` for IPv6-only, `inet` for IPv4-only, `any` for both (default). For the server, set it in `/etc/ssh/sshd_config` along with matching `ListenAddress` directives. For clients, configure in `~/.ssh/config` globally or per-host. Reload sshd with `systemctl reload sshd` after server changes. Verify with `sshd -T | grep addressfamily`. Use `ssh -o "AddressFamily inet6"` to override per-connection.
