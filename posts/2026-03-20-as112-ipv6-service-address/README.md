# How to Understand the AS112 IPv6 Service Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, AS112, DNS, Reverse DNS, RFC 7534, Networking

Description: Understand the AS112 project's IPv6 service addresses used to sink reverse DNS queries for private address ranges that would otherwise flood the root DNS servers.

## Introduction

AS112 is a distributed anycast DNS sinkhole that handles reverse DNS queries for private address ranges. Without AS112, reverse DNS lookups for RFC 1918 IPv4 addresses and RFC 4193 IPv6 ULA addresses would reach the internet's root nameservers unnecessarily.

## AS112 IPv6 Service Addresses

AS112 has two dedicated IPv6 addresses defined in RFC 7534:

```
Service Address 1: 2001:4:112::  (anycast)
Service Address 2: 2620:4f:8000:: (anycast)
```

These are announced via BGP from AS112 nodes worldwide.

## Why AS112 Matters

```bash
# Without AS112: a query for fc00::1 PTR would go to root servers
# With AS112: the query goes to a local AS112 node and returns NXDOMAIN

# Test AS112 is responding for ULA PTR queries
dig -x fc00::1 @2001:4:112::
# Should return NXDOMAIN quickly (from nearest AS112 node)

# Check if your resolver uses AS112
dig PTR 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.c.f.ip6.arpa.
```

## Zones Handled by AS112

```
IPv6 reverse zones delegated to AS112:
  8.e.f.ip6.arpa.  — fe80::/10 (link-local)
  9.e.f.ip6.arpa.  — fe90::/12
  a.e.f.ip6.arpa.  — fea0::/11
  b.e.f.ip6.arpa.  — feb0::/11 (additional link-local)
  c.f.ip6.arpa.    — fc00::/7 (ULA)
  d.f.ip6.arpa.    — fd00::/8 (ULA)
```

## Running Your Own AS112 Node (DNAME delegation)

```bash
# BIND zone for ULA reverse delegation to AS112
# /etc/bind/named.conf.local
zone "c.f.ip6.arpa" {
    type forward;
    forwarders { 2001:4:112::; 2620:4f:8000::; };
};

zone "d.f.ip6.arpa" {
    type forward;
    forwarders { 2001:4:112::; 2620:4f:8000::; };
};
```

## Monitoring AS112

```bash
# Verify AS112 connectivity
ping6 -c 3 2001:4:112::
ping6 -c 3 2620:4f:8000::

# Test PTR query response time
time dig PTR \
  1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.d.f.ip6.arpa
```

## Conclusion

AS112 prevents reverse DNS leakage for private IPv6 address ranges by providing a distributed anycast sinkhole. Configure your resolvers to forward appropriate reverse zones to AS112. Monitor AS112 reachability with OneUptime to ensure reverse DNS performance remains acceptable.
