# How to Troubleshoot HAProxy DNS Resolution Defaulting to IPv6 Instead of IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: HAProxy, DNS, IPv4, IPv6, Troubleshooting, Resolver, Networking

Description: Learn how to diagnose and fix HAProxy DNS resolution defaulting to IPv6 addresses instead of IPv4, causing connection failures to IPv4-only backends.

---

HAProxy can use DNS to resolve backend server hostnames dynamically. On dual-stack systems, the DNS resolver may return AAAA (IPv6) records first, causing HAProxy to attempt IPv6 connections to backends that only listen on IPv4.

## Symptoms

- HAProxy logs show `Connection refused` or `No route to host` errors.
- The resolved address looks like `::ffff:10.0.0.1` (IPv4-mapped IPv6) or a real IPv6 address.
- Backends are IPv4-only but hostnames resolve to both A and AAAA records.

## Checking What HAProxy Resolves

```bash
# Check what the system resolver returns for a hostname

getent hosts backend.internal

# Look in HAProxy runtime for resolved addresses
echo "show servers state" | socat stdio /var/run/haproxy/admin.sock | grep backend
```

## Fix 1: Use IPv4 Literals in the Backend

The simplest fix - bypass DNS entirely by using IP addresses.

```haproxy
backend app_servers
    server app1 10.0.0.1:8080 check
    server app2 10.0.0.2:8080 check
```

## Fix 2: Configure a Custom HAProxy Resolver with prefer-family

HAProxy's `resolvers` section supports a `prefer-family` directive to prefer A (IPv4) records.

```haproxy
# /etc/haproxy/haproxy.cfg

resolvers local_dns
    # Use your internal DNS server
    nameserver ns1 192.168.1.53:53

    # Prefer IPv4 (A) records over IPv6 (AAAA) records
    # Available in HAProxy 2.4+
    resolve-retries 3
    timeout resolve 1s
    timeout retry   1s
    hold valid 10s

backend app_servers
    balance roundrobin

    # Reference the resolver and enable dynamic DNS resolution
    server app1 backend-app1.internal:8080 check resolvers local_dns resolve-prefer ipv4
    server app2 backend-app2.internal:8080 check resolvers local_dns resolve-prefer ipv4
```

The `resolve-prefer ipv4` option on the `server` line tells HAProxy to prefer A records when both A and AAAA are returned.

## Fix 3: Configure the System Resolver to Prefer IPv4

Edit `/etc/gai.conf` to globally prefer IPv4 on the host:

```bash
# /etc/gai.conf
# Comment out or add this line to prefer IPv4
precedence ::ffff:0:0/96  100
```

This affects all applications on the host, not just HAProxy.

## Fix 4: Return Only A Records from DNS

If you control the DNS server, return only A records for internal hostnames:

```bash
# Example: BIND zone file entry - no AAAA record for backend
backend-app1.internal.  IN  A  10.0.0.1
```

## Verifying the Fix

```bash
# Confirm HAProxy resolves to an IPv4 address
echo "show servers state" | socat stdio /var/run/haproxy/admin.sock

# Test direct connection to the backend IPv4 address
curl -4 http://10.0.0.1:8080/health

# Reload HAProxy after config changes
systemctl reload haproxy
```

## Key Takeaways

- Use IP addresses instead of hostnames in HAProxy backends when DNS is unreliable.
- Use `resolve-prefer ipv4` on `server` lines with a `resolvers` block (HAProxy 2.4+).
- Edit `/etc/gai.conf` on the OS to prefer IPv4 system-wide as a fallback fix.
- Always verify resolved addresses via `show servers state` on the HAProxy admin socket.
