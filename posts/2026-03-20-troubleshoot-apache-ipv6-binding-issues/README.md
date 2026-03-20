# How to Troubleshoot Apache IPv6 Binding Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, Troubleshooting, Binding, Network Configuration

Description: Learn how to diagnose and fix Apache HTTP Server IPv6 binding issues, including address-in-use errors, failed binds, and configuration conflicts.

## Common Apache IPv6 Binding Errors

```bash
# Error 1: Address already in use
# AH00072: make_sock: could not bind to address [::]:80
# AH00455: Apache unable to open logs

# Error 2: Cannot assign requested address
# AH00072: make_sock: could not bind to address [2001:db8::10]:80

# Error 3: IPv6 not supported
# AH00548: NameVirtualHost has no VirtualHosts

# Check Apache error log
tail -50 /var/log/apache2/error.log
# or
journalctl -u apache2 -n 50
```

## Fix: Address Already in Use

```bash
# Find what's using port 80 on IPv6
ss -6 -tlnp | grep ':80'
fuser -n tcp 80

# If another process is using it
kill -9 <PID>

# Or if Apache is conflicting with itself
# (two processes trying to bind same port)
systemctl stop apache2
ss -6 -tlnp | grep ':80'  # Should be empty
systemctl start apache2
```

```apache
# Fix in Apache config: use proper dual-stack notation
# WRONG (may conflict on Linux with bindv6only=0):
Listen 80
Listen [::]:80

# RIGHT: Check bindv6only and configure accordingly
# If bindv6only=0 (Linux default): [::]:80 handles both
Listen [::]:80     # Handles IPv4 and IPv6 (when bindv6only=0)
# OR
Listen 0.0.0.0:80
Listen [::]:80     # Both explicit - should work with bindv6only=0 too
```

## Check bindv6only System Setting

```bash
# Check the system setting
cat /proc/sys/net/ipv6/bindv6only

# 0 = IPv6 socket can receive IPv4 connections
# 1 = IPv6 socket is IPv6-only

# If bindv6only=1:
# [::]:80 will NOT receive IPv4 connections
# You MUST use separate "Listen 80" for IPv4

# If bindv6only=0:
# [::]:80 may handle both, but explicit both directives also work
# Apache handles this internally
```

## Fix: Cannot Assign Requested Address

```bash
# The IPv6 address doesn't exist on the host
apache2ctl configtest
# Shows: make_sock: could not bind to address [2001:db8::10]:80

# Verify the address exists
ip -6 addr show | grep 2001:db8::10

# Add it if missing
ip -6 addr add 2001:db8::10/64 dev eth0

# Or change Listen to use all interfaces instead
# Listen [2001:db8::10]:80 → Listen [::]:80
```

## Verify Apache IPv6 Configuration

```bash
# Test configuration
apache2ctl configtest
apache2ctl -S    # Shows virtual host information

# Check listen addresses
apache2ctl -S 2>&1 | grep 'port\|address'

# Verify syntax of IPv6 in config files
grep -r 'Listen\|VirtualHost' /etc/apache2/ | grep -v '#'
# IPv6 addresses must be in brackets: [::]:80 or [2001:db8::10]:80
```

## Check if Apache Module Supports IPv6

```bash
# Check Apache was compiled with IPv6 support
apache2 -V | grep IPv6

# Output should include:
# Server built:   Mar 20 2026 ...
# Enable IPv6: yes

# Check loaded modules
apache2ctl -M | grep proxy
```

## Permission Issues

```bash
# Ports < 1024 require root or capabilities
# Apache usually starts as root, drops to www-data

# Check systemd service
systemctl status apache2

# Check Apache is starting as root
ps aux | grep apache2 | head -3
# First process should be root, workers www-data

# If permission denied:
# Check /etc/apache2/envvars or /etc/httpd/conf/httpd.conf
grep -E 'User|Group' /etc/apache2/envvars
```

## Quick Diagnostic Checklist

```bash
# 1. Config syntax
apache2ctl configtest

# 2. IPv6 addresses exist
ip -6 addr show

# 3. Port already in use
ss -6 -tlnp | grep ':80'

# 4. bindv6only setting
cat /proc/sys/net/ipv6/bindv6only

# 5. Firewall blocking
ip6tables -L INPUT -n | grep '80'
ufw status verbose | grep '80'

# 6. SELinux/AppArmor
getenforce   # SELinux
aa-status    # AppArmor
```

## Summary

Common Apache IPv6 binding issues: "Address already in use" — check `ss -6 -tlnp | grep ':80'` for port conflicts and verify `bindv6only` setting; "Cannot assign requested address" — the IPv6 address doesn't exist, add it with `ip -6 addr add`. Always use brackets in Apache config: `Listen [::]:80`. Check syntax with `apache2ctl configtest` and virtual host layout with `apache2ctl -S`.
