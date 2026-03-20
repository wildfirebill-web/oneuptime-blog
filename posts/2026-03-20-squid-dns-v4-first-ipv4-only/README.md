# How to Configure Squid DNS Lookups to Return IPv4 Only (dns_v4_first)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Squid, DNS, IPv4, Dns_v4_first, Configuration, Proxy, Networking

Description: Learn how to configure Squid to prefer or exclusively use IPv4 DNS resolution, preventing connections to IPv6 addresses on dual-stack networks.

---

On dual-stack networks, Squid may resolve hostnames to IPv6 addresses and attempt to connect to backends via IPv6, even when the preferred path is IPv4. The `dns_v4_first` directive and related settings control this behavior.

## The Problem

By default, Squid follows the system resolver's preference. On systems where IPv6 is enabled, DNS may return AAAA records first (per RFC 6724 address selection). If the IPv6 path is unreliable, connections fail or time out before falling back to IPv4.

## Fix 1: dns_v4_first on (Prefer IPv4)

```squid
# /etc/squid/squid.conf

# Prefer A (IPv4) records over AAAA (IPv6) records during DNS lookups

# Squid still falls back to IPv6 if no A record exists
dns_v4_first on
```

## Fix 2: Disable IPv6 Entirely in Squid

For environments where no IPv6 connectivity exists:

```squid
# Disable IPv6 socket binding (Squid won't create IPv6 sockets at all)
# This forces all outbound connections to use IPv4
dns_v4_first on

# Additionally, bind Squid to IPv4-only addresses
http_port 0.0.0.0:3128     # IPv4 only (not [::]:3128)
```

## Fix 3: System-Level IPv6 Preference (gai.conf)

This affects all applications, not just Squid:

```bash
# /etc/gai.conf
# Increase IPv4-mapped address precedence to prefer IPv4 connections
precedence ::ffff:0:0/96  100
```

## Fix 4: Use a DNS Server That Returns IPv4 Only

Point Squid at a DNS server configured to return only A records for your internal names.

```squid
# /etc/squid/squid.conf

# Use a specific DNS server (overrides /etc/resolv.conf)
dns_nameservers 192.168.1.53

# Optionally set the DNS query timeout
dns_timeout 30 seconds
```

## Verifying DNS Resolution

```bash
# Check what Squid resolves for a hostname
squidclient -h localhost -p 3128 mgr:dns | grep example.com

# Check the system-level resolution
getent hosts example.com

# Compare with explicit IPv4 lookup
dig +short A example.com @192.168.1.53
dig +short AAAA example.com @192.168.1.53
```

## Confirming IPv4 Outbound Connections

```bash
# Watch Squid's outbound connections in real time
# IPv4 connections show x.x.x.x:port; IPv6 shows [::]:port
ss -tnp | grep squid

# Check the access log for DIRECT connections and the IP used
tail -f /var/log/squid/access.log | grep DIRECT
```

## Key Takeaways

- `dns_v4_first on` makes Squid prefer A records but still falls back to AAAA.
- Bind `http_port` to `0.0.0.0` (not `::`) to prevent Squid from creating IPv6 listening sockets.
- For strict IPv4-only behavior, combine `dns_v4_first` with an IPv4-only DNS server.
- `/etc/gai.conf` provides a system-wide fallback that affects Squid and all other applications.
