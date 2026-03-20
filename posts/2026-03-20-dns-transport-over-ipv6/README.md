# How to Understand DNS Transport over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS, DNS Transport, UDP, TCP

Description: An explanation of how DNS query and response transport works over IPv6, including UDP vs TCP usage, port numbers, and IPv6-specific considerations.

## DNS Transport Basics

DNS uses UDP port 53 for most queries due to its low overhead. TCP port 53 is used for:
- Responses larger than 512 bytes (or 4096 bytes with EDNS0)
- Zone transfers (AXFR/IXFR)
- DNS-over-TLS (DoT) on port 853

These rules apply equally to IPv4 and IPv6. The transport protocol (IPv4 or IPv6) is independent of DNS query behavior.

## IPv6 DNS Query Format

A DNS query over IPv6 uses the same DNS message format as IPv4. The only differences are at the IP layer:

```
Outer IPv6 Header:
  Source: 2001:db8:client::1
  Destination: 2001:db8::53 (DNS server IPv6 address)
  Next Header: UDP (17) or TCP (6)

UDP Header:
  Source Port: 12345 (random ephemeral)
  Destination Port: 53

DNS Message: (identical format to IPv4)
  Query: AAAA www.example.com
```

## UDP vs TCP in IPv6 DNS

```bash
# Most DNS queries use UDP
# dig uses UDP by default
dig AAAA example.com @2001:4860:4860::8888

# Force TCP for DNS query
dig AAAA example.com @2001:4860:4860::8888 +tcp

# Check if TCP is being used (watch packet size)
dig AAAA example.com @::1 +stats | grep "MSG SIZE"
# If MSG SIZE > 512: might trigger TCP retry
# EDNS0 allows up to 4096 bytes over UDP
```

## EDNS0 and IPv6

EDNS0 (Extension Mechanisms for DNS) increases the UDP payload size limit from 512 to 4096 bytes. This is important for DNSSEC responses which can be large:

```bash
# Enable EDNS0 with 4096 byte buffer (default in modern dig)
dig AAAA example.com +bufsize=4096

# Check if EDNS0 is being used in responses
dig AAAA example.com | grep "EDNS"
# OPT PSEUDOSECTION with version and size info

# Disable EDNS0 (forces 512 byte limit)
dig AAAA example.com +noedns
```

## IPv6 Fragmentation and DNS

IPv6 does not fragment packets in transit (only at the source). Large DNS responses over UDP may require fragmentation, but IPv6 fragmentation is unreliable in practice because many firewalls drop fragmented IPv6 packets.

The recommended approach:
- Use EDNS0 buffer size of 1232 bytes (to stay within typical IPv6 MTU) for UDP
- Fall back to TCP for responses that exceed the buffer size

```bash
# Use conservative EDNS0 buffer size (RIPE recommendation: 1232)
dig AAAA example.com +bufsize=1232

# Check if DNS server falls back to TCP for large responses
dig DNSKEY example.com +dnssec | grep "flags.*TC"
# TC (truncated) flag means the response was too large for UDP
# Client should retry over TCP automatically
```

## IPv6 Source Address Selection for DNS Queries

When a client has multiple IPv6 addresses, the OS uses RFC 6724 address selection to pick the source:

```bash
# Check which source address is used for DNS queries
tcpdump -i eth0 -n 'port 53' &
dig AAAA google.com @2001:4860:4860::8888
fg; # Ctrl+C

# The source IPv6 address in the captured packets shows what was selected
```

## DNS over IPv6 with Link-Local Addresses

Link-local addresses require specifying the interface:

```bash
# Query DNS server at link-local address (must specify interface with %)
dig AAAA example.com @fe80::1%eth0

# BIND listening on link-local
# In named.conf:
# listen-on-v6 { fe80::1; };
# (Not recommended for production - use global addresses)
```

## Verifying Full DNS Stack over IPv6

```bash
# Complete end-to-end test: client → resolver → authoritative → response

# 1. Client to recursive resolver over IPv6
dig AAAA example.com @2001:db8::recursive

# 2. Recursive resolver to authoritative over IPv6
# Enable query logging on resolver to verify
# Unbound: verbosity 3
# BIND: rndc querylog on

# 3. Check the resolver's outbound interface
# It should use a global IPv6 address, not link-local
```

## MTU and Fragmentation Issues with IPv6 DNS

```bash
# Test if large DNS responses work (DNSSEC adds signature records)
dig DNSKEY . @2001:4860:4860::8888 +dnssec +bufsize=4096

# If this fails but small queries work: fragmentation problem
# Check if fragmented IPv6 is being dropped
ping6 -M do -s 1400 2001:4860:4860::8888

# Workaround: reduce EDNS0 buffer size to avoid fragmentation
# /etc/unbound/unbound.conf:
# edns-buffer-size: 1232
```

## Summary

DNS transport over IPv6 uses the same DNS message format as IPv4, just with IPv6 source and destination addresses. UDP port 53 handles most queries, TCP port 53 for large responses and zone transfers. Key IPv6-specific concerns include fragmentation (use conservative EDNS0 buffer size of 1232 bytes) and source address selection (RFC 6724 determines which IPv6 address is used for queries). Always allow ICMPv6 through firewalls to enable proper MTU path discovery for DNS.
