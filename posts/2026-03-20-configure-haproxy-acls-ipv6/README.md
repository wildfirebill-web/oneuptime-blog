# How to Configure HAProxy ACLs for IPv6 Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, HAProxy, ACL, Access Control, Load Balancer

Description: Learn how to configure HAProxy ACLs (Access Control Lists) for IPv6 addresses and subnets to route traffic, restrict access, and implement IPv6-aware load balancing policies.

## Basic IPv6 ACLs

```haproxy
frontend http_front
    bind *:80
    bind [::]:80

    # ACL matching a specific IPv6 address
    acl is_admin_host src 2001:db8::admin

    # ACL matching an IPv6 subnet
    acl is_internal_ipv6 src 2001:db8:internal::/48

    # ACL matching multiple prefixes
    acl is_trusted src 2001:db8:trusted::/48 fd00:corp::/32

    # Route based on ACL
    use_backend admin_backend if is_admin_host
    use_backend internal_backend if is_internal_ipv6
    default_backend public_backend
```

## Block IPv6 Addresses

```haproxy
frontend http_front
    bind *:80
    bind [::]:80

    # Block specific IPv6 hosts
    acl is_blocked src 2001:db8::bad:actor 2001:db8::another:bad:host

    # Block entire subnet
    acl is_blocked_subnet src 2001:db8:blocked::/48

    # Return 403 for blocked addresses
    http-request deny status 403 if is_blocked || is_blocked_subnet

    default_backend app_servers
```

## Rate Limiting Based on IPv6 Address

```haproxy
# Define a stick table for rate limiting

frontend http_front
    bind *:80
    bind [::]:80

    # Create a stick table keyed by source IPv6 address
    # Rate limit: 100 requests per 10 seconds per IP
    stick-table type ipv6 size 100k expire 10s store http_req_rate(10s)
    http-request track-sc0 src

    # Deny if rate exceeded
    acl too_many_requests sc_http_req_rate(0) gt 100
    http-request deny status 429 if too_many_requests

    default_backend app_servers
```

## IPv6 Subnet Routing

```haproxy
frontend http_front
    bind *:80
    bind [::]:80

    # Route different IPv6 subnets to different backends
    acl is_eu_ipv6 src 2001:db8:eu::/48
    acl is_us_ipv6 src 2001:db8:us::/48
    acl is_asia_ipv6 src 2001:db8:asia::/48

    use_backend eu_servers if is_eu_ipv6
    use_backend us_servers if is_us_ipv6
    use_backend asia_servers if is_asia_ipv6
    default_backend global_servers
```

## ACL from File for Large IPv6 Lists

```haproxy
frontend http_front
    bind *:80
    bind [::]:80

    # Load IPv6 address/subnet list from file
    # /etc/haproxy/ipv6-whitelist.txt
    acl trusted_ipv6 src -f /etc/haproxy/ipv6-whitelist.txt

    # Require authentication for non-trusted IPv6
    http-request deny if !trusted_ipv6

    default_backend app_servers
```

```bash
# /etc/haproxy/ipv6-whitelist.txt
# One IPv6 address or subnet per line
cat > /etc/haproxy/ipv6-whitelist.txt << 'EOF'
2001:db8:trusted::/48
2001:db8::admin
fd00:corp::/32
::1
EOF
```

## Combining IPv4 and IPv6 ACLs

```haproxy
frontend http_front
    bind *:80
    bind [::]:80

    # Allow access from both IPv4 and IPv6 internal ranges
    acl is_internal_v4 src 192.168.0.0/16 10.0.0.0/8
    acl is_internal_v6 src 2001:db8:internal::/48 fd00::/8

    acl is_internal src is_internal_v4 is_internal_v6

    # Admin area only for internal
    acl is_admin_path path_beg /admin

    http-request deny if is_admin_path !is_internal

    default_backend app_servers
```

## Summary

HAProxy ACLs for IPv6 use `acl name src 2001:db8::/32` for subnet matching and `acl name src 2001:db8::addr` for specific addresses. Use `http-request deny` to block, `use_backend` to route, and stick tables for rate limiting. Load large IPv6 lists with `src -f /path/to/file`. Combine IPv4 and IPv6 ACLs with multiple `src` entries or separate ACLs in the same `use_backend` condition.
