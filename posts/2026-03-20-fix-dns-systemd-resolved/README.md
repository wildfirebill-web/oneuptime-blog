# How to Fix DNS Resolution Errors with systemd-resolved

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, systemd-resolved, Linux, Troubleshooting, Configuration, Ubuntu

Description: Diagnose and fix DNS resolution failures caused by systemd-resolved misconfiguration, broken /etc/resolv.conf symlinks, and DNS server connectivity issues.

## Introduction

`systemd-resolved` is the default DNS stub resolver on Ubuntu, Debian, Fedora, and other modern Linux distributions. It handles DNS caching, DNSSEC validation, and DNS-over-TLS. Misconfiguration or service failures cause all DNS resolution to stop working for applications on the system. This guide covers diagnosing and fixing the most common systemd-resolved problems.

## Quick Diagnosis

```bash
# Check if systemd-resolved is running:
systemctl status systemd-resolved

# Check current DNS configuration and status:
resolvectl status
# Shows: per-interface DNS settings, current DNS server, DNS over TLS status

# Test if it's working:
resolvectl query google.com
# If this works but dig doesn't: /etc/resolv.conf issue
# If this fails: systemd-resolved internal issue

# Check for error messages:
journalctl -u systemd-resolved --since "1 hour ago" | tail -30
```

## Fix: Broken /etc/resolv.conf Symlink

```bash
# Most common issue: /etc/resolv.conf not pointing to systemd-resolved

# Check current state:
ls -la /etc/resolv.conf
# Should be a symlink to: /run/systemd/resolve/stub-resolv.conf
# Or contain: nameserver 127.0.0.53

# If not a symlink, check what's there:
cat /etc/resolv.conf

# Fix by creating correct symlink:
# (WARNING: this overwrites current file)
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

# Alternative: write static content pointing to resolved stub:
cat > /etc/resolv.conf << 'EOF'
# This file is managed by systemd-resolved
nameserver 127.0.0.53
options edns0 trust-ad
search .
EOF

# Verify it works:
dig google.com
```

## Fix: systemd-resolved Not Starting

```bash
# Check why it's not starting:
systemctl start systemd-resolved
journalctl -u systemd-resolved -n 50

# Common causes:
# Port 53 conflict (another process using port 53):
ss -ulnp | grep :53
# If something else is using 53: conflict with systemd-resolved's stub listener

# Disable conflicting DNS:
# If dnsmasq or bind is also running:
systemctl stop dnsmasq
systemctl disable dnsmasq

# Or: disable systemd-resolved's stub listener if you want to use another DNS:
# /etc/systemd/resolved.conf:
# [Resolve]
# DNSStubListener=no

systemctl restart systemd-resolved
```

## Fix: Wrong DNS Server

```bash
# Check which DNS server is being used:
resolvectl status | grep "Current DNS Server"

# Configure specific DNS servers:
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=9.9.9.9
Domains=~.   # Use for all domains
EOF

systemctl restart systemd-resolved

# Per-interface DNS (overrides global):
resolvectl dns eth0 10.20.0.1 10.20.0.2
resolvectl domain eth0 company.internal
# Or permanently via /etc/systemd/network/eth0.network

# Verify:
resolvectl status eth0
```

## Fix: DNSSEC Validation Failures

```bash
# DNSSEC can cause SERVFAIL for valid domains if resolver has clock skew
# or if the domain's DNSSEC is broken:

# Check DNSSEC setting:
resolvectl status | grep -i dnssec

# Disable DNSSEC if causing failures:
cat >> /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNSSEC=no
EOF
systemctl restart systemd-resolved

# Test with DNSSEC disabled:
resolvectl query example.com

# Re-enable after investigating the failing domain:
# DNSSEC=allow-downgrade  # More lenient
# DNSSEC=yes              # Strict (fail if DNSSEC is broken)
```

## Fix: DNS Resolution in Containers

```bash
# Docker containers may not use systemd-resolved automatically
# Check Docker's DNS configuration:
cat /etc/docker/daemon.json
# If empty or no "dns" key: Docker uses host's resolv.conf

# Tell Docker to use the systemd-resolved stub:
cat > /etc/docker/daemon.json << 'EOF'
{
  "dns": ["127.0.0.53"],
  "dns-search": []
}
EOF
systemctl restart docker
```

## Monitor and Flush Cache

```bash
# View DNS statistics (cache hit rate, etc.):
resolvectl statistics

# Flush all DNS caches:
resolvectl flush-caches

# Watch DNS queries in real-time (enable logging):
resolvectl log-level debug
journalctl -u systemd-resolved -f &
dig google.com
resolvectl log-level info  # Restore to normal

# Check DNSSEC validation status for a domain:
resolvectl query --type=DNSSEC example.com
```

## Conclusion

systemd-resolved issues typically fall into three categories: broken `/etc/resolv.conf` symlink (fix with `ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`), wrong DNS server configured (fix in `/etc/systemd/resolved.conf`), and DNSSEC validation failures (disable with `DNSSEC=no` while investigating). Use `resolvectl status` to see the current effective configuration and `resolvectl statistics` to monitor cache performance. For persistent configuration, always edit `/etc/systemd/resolved.conf` and restart the service.
