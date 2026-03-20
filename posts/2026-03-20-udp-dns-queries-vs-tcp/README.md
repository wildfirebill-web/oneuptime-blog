# How to Use UDP for DNS Queries vs TCP Fallback

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, DNS, TCP, Protocol, Networking, Linux

Description: Understand when DNS uses UDP versus TCP, how the fallback mechanism works, and how to force DNS queries over TCP for testing and troubleshooting.

## Introduction

DNS primarily uses UDP for its query/response model — a single 512-byte UDP packet is sufficient for most queries. TCP is used as a fallback when responses exceed 512 bytes (or 4096 bytes with EDNS0), when zone transfers are requested, or when UDP connectivity is unreliable. Understanding this distinction helps debug DNS failures and test DNS behavior in both transport modes.

## When DNS Uses UDP vs TCP

```
DNS uses UDP when:
  - Response fits in 512 bytes (traditional limit)
  - Response fits in EDNS0 advertised buffer size (typically 4096 bytes)
  - Standard query/response (A, AAAA, MX, etc.)
  - Client and server both support EDNS0

DNS uses TCP when:
  - Response exceeds UDP payload limit (TC bit set in UDP response)
  - Zone transfer requested (AXFR, IXFR)
  - Some DNSSEC responses with many records
  - DNS-over-TLS (DoT) always uses TCP
  - Resolver configured with TCP-only setting
  - UDP is blocked by firewall (TCP as fallback)
```

## Default UDP Behavior

```bash
# DNS query using UDP (default):
dig google.com
# Shows: ;; Query time: X msec
# Typically uses UDP port 53

# Verify UDP was used:
dig google.com +stats | grep 'SERVER:'
# Connection is established to :53 via UDP by default

# Capture to verify:
sudo tcpdump -i eth0 -n 'udp port 53 or tcp port 53' &
dig google.com
# Should see: UDP packets only (unless response is large)
```

## Forcing DNS Over TCP

```bash
# Force TCP for a single dig query:
dig google.com +tcp
# +tcp flag forces TCP even if response fits in UDP

# Verify TCP is used:
sudo tcpdump -i eth0 -n 'tcp port 53' &
dig google.com +tcp
# Should see: TCP 3-way handshake + DNS query over TCP

# Force TCP in /etc/resolv.conf (all system DNS):
# This is a non-standard extension, but some resolvers support it
# Typically done via stub resolver configuration instead

# Force TCP in systemd-resolved:
# /etc/systemd/resolved.conf:
# [Resolve]
# DNS=8.8.8.8
# DNSOverTLS=yes  # Uses DoT (TCP-based TLS)
```

## The TC (Truncated) Bit Mechanism

```bash
# When UDP response is too large:
# 1. Server sets TC bit in UDP response
# 2. Client retries the same query over TCP
# 3. Server sends full response over TCP

# Simulate by querying a domain with many records:
dig ANY google.com
# Large responses may trigger TC bit + TCP retry

# Check if TC bit was set:
dig ANY google.com +noadditional | grep -i "ANSWER\|truncated"

# See the TC bit in dig output:
dig google.com @8.8.8.8 +bufsize=0
# bufsize=0: tell server we want very small UDP responses
# Server should truncate and set TC bit
# dig will then retry over TCP

# In Wireshark:
# Filter: dns
# Look for: dns.flags.tc == 1 (truncated bit set)
```

## EDNS0 and Large UDP Responses

```bash
# EDNS0 extends DNS to allow larger UDP payloads (default 4096 bytes):
dig google.com +edns=0 +bufsize=4096
# Shows: ; EDNS: version: 0, flags: do; udp: 4096 (advertising 4096 byte buffer)

# Check if EDNS0 is supported by a server:
dig +ednsopt @8.8.8.8 google.com
# Server should respond with its EDNS0 buffer size

# Disable EDNS0 to force 512-byte limit (old behavior):
dig google.com +noedns
```

## Testing DNS TCP Fallback

```bash
# Test that your resolver works when UDP is blocked:
# Block UDP DNS temporarily:
iptables -A OUTPUT -p udp --dport 53 -j DROP

# DNS should fall back to TCP:
dig google.com   # Should still work (via TCP)

# Remove the block:
iptables -D OUTPUT -p udp --dport 53 -j DROP

# Test with nmap to verify both UDP and TCP ports are open:
nmap -sU -sS -p 53 8.8.8.8
# Both UDP and TCP port 53 should show open for a full resolver
```

## Conclusion

DNS uses UDP by default for efficiency — the 8-byte UDP header versus 20-byte TCP header matters when handling millions of queries. TCP is the fallback for large responses (TC bit mechanism) and mandatory for zone transfers. Always ensure your DNS servers and firewalls allow both UDP/53 and TCP/53. Use `dig +tcp` to test DNS over TCP directly, and monitor for TC bit responses in your DNS traffic — many TC responses indicate your clients need EDNS0 buffer size increases.
