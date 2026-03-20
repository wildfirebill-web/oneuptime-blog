# How to Set Up Unbound DNS Resolver for IPv4 on OPNsense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OPNsense, Unbound, DNS, IPv4, DNS Resolver, Security, DNSSEC

Description: Configure the Unbound DNS resolver on OPNsense for IPv4 caching with DNSSEC validation, host overrides for local names, and DNS-over-TLS for encrypted upstream queries.

## Introduction

OPNsense uses Unbound as its default DNS resolver. Unlike a simple forwarder, Unbound performs full recursive resolution by querying authoritative servers directly, optionally validates DNSSEC, and caches results for LAN clients.

## Basic Unbound Configuration

Navigate to **Services > Unbound DNS > General**:

```
Enable:                  checked
Listen Port:             53
Network Interfaces:      LAN, Loopback
DNSSEC:                  checked
DHCP Registration:       checked  (auto-register DHCP leases)
Static DHCP:             checked  (register static mappings)
OpenNIC:                 unchecked
Register ISP-provided nameservers: unchecked (use root servers)

Advanced:
  Prefetch Support:      checked   (prefetch popular records)
  Cache Min TTL:         300
  Cache Max TTL:         86400
```

## Host Overrides (Local DNS)

Navigate to **Services > Unbound DNS > Host Overrides > Add**:

```
Host:        fileserver
Domain:      local.lan
IP:          192.168.1.10
Description: File Server

Host:        nas
Domain:      local.lan
IP:          192.168.1.50
```

## Domain Override (Forward Specific Domains)

Navigate to **Services > Unbound DNS > Domain Overrides > Add**:

```
Domain:      corp.internal
IP:          10.1.1.10   (internal DNS server for .corp.internal)
Description: Internal corporate domain
```

## DNS-over-TLS (DoT)

Navigate to **Services > Unbound DNS > DNS over TLS**:

```
Add upstream:
  Server:  1.1.1.1
  Port:    853
  TLS hostname: cloudflare-dns.com

  Server:  9.9.9.9
  Port:    853
  TLS hostname: dns.quad9.net
```

## Access Control Lists

Navigate to **Services > Unbound DNS > Access Lists > Add**:

```
Access List Name:    Internal
Networks:
  Action: Allow  Network: 192.168.1.0/24
  Action: Allow  Network: 10.1.0.0/16
  Action: Refuse Network: 0.0.0.0/0   (deny all others)
```

## Verify Unbound

```bash
# OPNsense CLI (via SSH or console)
dig @192.168.1.1 google.com

# Check DNSSEC validation
dig @192.168.1.1 google.com +dnssec

# Test local override
dig @192.168.1.1 fileserver.local.lan

# Check Unbound status
unbound-control status

# Flush cache
unbound-control flush_zone .
```

## Conclusion

Unbound on OPNsense provides DNSSEC-validating, recursive DNS resolution for IPv4 clients. Enable DHCP registration for automatic local name resolution, add host overrides for static servers, configure domain overrides for internal domains, and use DNS-over-TLS for encrypted upstream queries. Always point DHCP clients to the OPNsense IP as their DNS server.
