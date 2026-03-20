# How to Configure HAProxy Dual-Stack with v4v6 Option

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, HAProxy, Dual-Stack, v4v6, Load Balancer

Description: Learn how to configure HAProxy dual-stack mode using the v4v6 bind option to accept both IPv4 and IPv6 connections on a single socket, and when to use separate bind directives instead.

## The v4v6 Bind Option

HAProxy provides a `v4v6` option that makes a single IPv6 socket accept IPv4-mapped connections (similar to Linux's `net.ipv6.bindv6only=0`):

```haproxy
frontend http_v4v6
    # Single socket handles both IPv4 and IPv6
    bind [::]:80 v4v6

    default_backend app_servers
```

With `v4v6`:
- IPv4 clients appear as `::ffff:192.168.1.1` (IPv4-mapped)
- IPv6 clients appear as `2001:db8::10`

## Separate Binds (Recommended for Clarity)

```haproxy
frontend http_dual_stack
    # Explicit dual-stack: separate sockets
    bind *:80
    bind [::]:80

    default_backend app_servers
```

With separate binds:
- IPv4 clients have proper `192.168.1.1` address
- IPv6 clients have proper `2001:db8::10` address
- Cleaner ACLs and logging

## v4v6 for HTTPS Dual-Stack

```haproxy
frontend https_v4v6
    # Single HTTPS socket for both versions
    bind [::]:443 v4v6 ssl crt /etc/ssl/haproxy/example.pem

    ssl-default-bind-options ssl-min-ver TLSv1.2
    mode http

    default_backend secure_backend
```

## ACLs with v4v6 (IPv4-Mapped Addresses)

```haproxy
frontend http_v4v6
    bind [::]:80 v4v6

    # With v4v6, IPv4 addresses appear as ::ffff:A.B.C.D
    acl is_ipv4_mapped src ::ffff:0.0.0.0/96
    acl is_pure_ipv6 src 2001:db8::/32

    # Internal IPv4 range (in IPv4-mapped format)
    acl is_internal_v4 src ::ffff:192.168.0.0/112
    # ::ffff:192.168.0.0/112 = ::ffff:192.168.0.0 to ::ffff:192.168.255.255

    use_backend internal_backend if is_internal_v4
    default_backend public_backend
```

## Separate Binds with ACLs (Cleaner)

```haproxy
frontend http_dual
    bind *:80
    bind [::]:80

    # ACLs work with real addresses
    acl is_ipv6 src 2001:db8::/32
    acl is_ipv4_internal src 192.168.0.0/16

    use_backend internal if is_ipv4_internal
    use_backend ipv6_only_backend if is_ipv6
    default_backend public_backend
```

## When to Use v4v6 vs Separate Binds

| Scenario | Use v4v6 | Use Separate Binds |
|----------|----------|--------------------|
| Simple setup | Yes | Yes |
| ACLs based on source IP | Avoid | Recommended |
| Logging real IPs | Complex | Simple |
| System-level bindv6only=1 | Won't work | Use separate |
| Prefer simplicity | Yes | - |

## Test Dual-Stack Configuration

```bash
# Verify HAProxy config
haproxy -c -f /etc/haproxy/haproxy.cfg

# Test IPv4 connection
curl -4 http://example.com/

# Test IPv6 connection
curl -6 http://[2001:db8::10]/

# Check log for correct IP capture
tail -f /var/log/haproxy.log | grep -E '\[.*:.*\]|::ffff'
```

## Summary

HAProxy provides two approaches for dual-stack: `bind [::]:80 v4v6` (single socket for both, IPv4 appears as `::ffff:A.B.C.D`), or separate `bind *:80` and `bind [::]:80` (recommended for clean IP handling in ACLs and logs). Use `v4v6` for simple setups; use separate binds when you need IP-based ACLs or clean logging. Both methods work for HTTPS by appending `ssl crt /path/to/cert.pem`.
