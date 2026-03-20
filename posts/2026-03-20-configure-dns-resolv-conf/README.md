# How to Configure DNS Servers in /etc/resolv.conf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Linux, resolv.conf, Configuration, Networking, Resolver

Description: Configure DNS resolver settings in /etc/resolv.conf including multiple nameservers, search domains, and timeout options for reliable DNS resolution.

## Introduction

`/etc/resolv.conf` is the traditional configuration file for the Linux DNS stub resolver. It specifies which DNS servers to query and in what order, along with search domain suffixes and timeout behaviors. On modern systems with `systemd-resolved` or `NetworkManager`, this file is often managed automatically, but understanding and manually configuring it remains essential for servers and containers.

## Basic Configuration

```bash
# Minimal /etc/resolv.conf with two nameservers:
cat > /etc/resolv.conf << 'EOF'
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF

# With organization's DNS plus public fallback:
cat > /etc/resolv.conf << 'EOF'
nameserver 10.20.0.1     # Internal DNS (primary)
nameserver 10.20.0.2     # Internal DNS (secondary)
nameserver 8.8.8.8       # Public fallback
EOF
# Note: Up to 3 nameservers supported; additional entries are ignored
```

## All Configuration Options

```
# /etc/resolv.conf complete reference:

nameserver <IP>
  # DNS server to query. Up to 3 allowed.
  # Queried in order; if first doesn't respond, try next.

domain <domain>
  # Local domain name. Sets single search domain.
  # Can't be used with 'search' simultaneously.

search <domain1> [domain2] [domain3]
  # Search list for short hostname lookups.
  # "ping db" tries: db.domain1, db.domain2, db.domain3, db
  # Up to 6 domains; total 256 characters.

options timeout:<n>
  # Seconds to wait for each nameserver response. Default: 5.
  # Lower (1-2) for faster failover.

options attempts:<n>
  # Number of retries per nameserver. Default: 2.
  # Lower for faster failover.

options rotate
  # Rotate through nameservers for load balancing.
  # Default: always try first server first.

options ndots:<n>
  # Minimum dots before treating as absolute (not searching).
  # Default: 1. api.example.com (1 dot) = try search first.
  # With ndots:2, api.example.com is tried absolutely first.
```

## Example Configurations

```bash
# Enterprise with internal DNS and fast failover:
cat > /etc/resolv.conf << 'EOF'
nameserver 10.20.0.10
nameserver 10.20.0.11
nameserver 8.8.8.8
search company.internal us.company.internal
domain company.internal
options timeout:2 attempts:2
EOF

# Container/Docker environment (single resolver):
cat > /etc/resolv.conf << 'EOF'
nameserver 172.17.0.1     # Docker bridge gateway
options ndots:5 timeout:3
EOF

# Kubernetes pod DNS:
cat > /etc/resolv.conf << 'EOF'
nameserver 10.96.0.10     # kube-dns service IP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
EOF
# ndots:5: Kubernetes requires this for proper in-cluster resolution
```

## Protect Against Overwriting

```bash
# NetworkManager and DHCP often overwrite /etc/resolv.conf
# Methods to prevent:

# Method 1: Make file immutable (root can't overwrite):
chattr +i /etc/resolv.conf
# Undo with: chattr -i /etc/resolv.conf

# Method 2: NetworkManager - configure to not manage DNS:
# /etc/NetworkManager/NetworkManager.conf:
# [main]
# dns=none    ← NM won't touch resolv.conf

# Method 3: Link to your managed file:
# Create your config elsewhere and symlink:
cp /etc/resolv.conf /etc/resolv.conf.manual
ln -sf /etc/resolv.conf.manual /etc/resolv.conf
# Then protect the target:
chattr +i /etc/resolv.conf.manual
```

## Verify Configuration Works

```bash
# Test resolution with configured servers:
dig google.com
# Should use nameservers from resolv.conf

# Test search domain behavior:
# With 'search company.internal':
ping db         # Tries db.company.internal first
getent hosts db  # Shows what resolves to

# Check which resolver is actually used:
strace -e openat dig google.com 2>&1 | grep resolv
# Shows if resolv.conf is being read

# Debug resolver behavior:
RESOLV_HOST_CONF=/etc/resolv.conf RESOLV_CONF=/etc/resolv.conf \
  nscd -d -f  # nscd debug mode (if installed)
```

## Conclusion

`/etc/resolv.conf` controls which DNS servers Linux queries and how. Always specify 2-3 nameservers for redundancy. Use the `search` directive to allow short hostnames in Kubernetes and corporate environments. Set `options timeout:2 attempts:2` for faster failover when the primary DNS server is unreachable. On systemd-resolved systems, `resolvectl status` shows the effective DNS configuration regardless of how `/etc/resolv.conf` is managed.
