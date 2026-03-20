# How to Troubleshoot Nginx IPv6 Binding Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Nginx, Troubleshooting, Binding Errors, Configuration

Description: Learn how to diagnose and fix common Nginx IPv6 binding errors, including 'Address already in use', 'bind() failed', and configuration conflicts.

## Common Nginx IPv6 Binding Errors

```bash
# Error 1: Address already in use

# nginx: [emerg] bind() to [::]:80 failed (98: Address already in use)

# Error 2: Cannot bind on socket
# nginx: [emerg] bind() to [2001:db8::10]:80 failed (99: Cannot assign requested address)

# Error 3: Permission denied (ports < 1024 without root)
# nginx: [emerg] bind() to [::]:80 failed (13: Permission denied)
```

## Fix: Address Already in Use (EADDRINUSE)

This occurs when `[::]:80` and `0.0.0.0:80` conflict because `bindv6only=0`:

```bash
# Check the system setting
cat /proc/sys/net/ipv6/bindv6only

# If 0: [::] can receive IPv4 connections, conflicting with 0.0.0.0

# Fix 1: Use ipv6only=on in nginx configuration
server {
    listen 80;                    # IPv4: 0.0.0.0:80
    listen [::]:80 ipv6only=on;   # IPv6: [::]:80 (IPv6 only)
}
```

```nginx
# Fix 2: Use only [::] without 0.0.0.0 (if ipv6only=off)
server {
    listen [::]:80;   # Handles both IPv4 and IPv6 (when ipv6only=0)
}
```

## Fix: Cannot Assign Requested Address (EADDRNOTAVAIL)

```bash
# This error means the IPv6 address doesn't exist on the host
# nginx: [emerg] bind() to [2001:db8::10]:80 failed (99: Cannot assign requested address)

# Check if the address exists
ip -6 addr show

# If address is missing, add it:
ip -6 addr add 2001:db8::10/64 dev eth0

# Or check nginx config for typos
nginx -T | grep 'listen.*:.*\]'

# Fix: Use [::] to listen on all IPv6 addresses instead of specific one
server {
    listen [::]:80 ipv6only=on;   # All IPv6 interfaces
}
```

## Fix: Permission Denied (EACCES)

```bash
# Ports < 1024 require root or CAP_NET_BIND_SERVICE
# nginx: [emerg] bind() to [::]:80 failed (13: Permission denied)

# Fix 1: Run nginx as root (systemd handles this)
systemctl status nginx

# Fix 2: Grant capability to nginx binary
setcap 'cap_net_bind_service=+ep' /usr/sbin/nginx

# Fix 3: Use nginx systemd override
mkdir -p /etc/systemd/system/nginx.service.d/
cat > /etc/systemd/system/nginx.service.d/override.conf << 'EOF'
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
EOF
systemctl daemon-reload
systemctl restart nginx
```

## Diagnostic Commands

```bash
# Test nginx configuration syntax
nginx -t

# Show full nginx configuration (merged)
nginx -T

# Check what's already listening on port 80 IPv6
ss -6 -tlnp | grep ':80'

# Check if IPv6 is available on the host
ip -6 addr show

# Check the bindv6only setting
sysctl net.ipv6.bindv6only

# Check systemd journal for nginx errors
journalctl -u nginx -n 50 --no-pager
```

## Validate IPv6 Configuration

```bash
# Common mistakes:
# 1. Missing brackets around IPv6 address
# WRONG: listen 2001:db8::10:80;    (nginx interprets :: as part of hostname)
# RIGHT: listen [2001:db8::10]:80;

# 2. Missing ipv6only when using both [::] and 0.0.0.0
# WRONG:
server {
    listen 80;
    listen [::]:80;           # Missing ipv6only=on
}
# RIGHT:
server {
    listen 80;
    listen [::]:80 ipv6only=on;
}

# Check config for proper bracket notation
grep 'listen' /etc/nginx/sites-enabled/* | grep -v '\[' | grep ':'
```

## Quick Reference: Common Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| EADDRINUSE | [::] conflicts with 0.0.0.0 | Add `ipv6only=on` |
| EADDRNOTAVAIL | IPv6 address not configured | Add address to interface |
| EACCES | Port < 1024, not root | Use setcap or systemd |
| EAFNOSUPPORT | IPv6 disabled in kernel | Enable IPv6 |

```bash
# Check if IPv6 is disabled at kernel level
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
# 0 = enabled, 1 = disabled
```

## Summary

Common Nginx IPv6 binding errors: "Address already in use" - fix with `ipv6only=on` on the `[::]:PORT` listener; "Cannot assign requested address" - the IPv6 address doesn't exist on the host, check with `ip -6 addr show`; "Permission denied" - use `setcap cap_net_bind_service`. Always use brackets around IPv6 addresses in nginx config: `listen [2001:db8::10]:80;`. Diagnose with `nginx -t` and `ss -6 -tlnp`.
