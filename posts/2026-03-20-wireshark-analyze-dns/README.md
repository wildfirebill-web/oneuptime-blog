# How to Analyze DNS Queries and Responses in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, DNS, IPv4, Networking, Diagnostics, Packet Analysis

Description: Use Wireshark to capture and analyze DNS query and response packets, view resolved addresses, identify failures, and measure DNS lookup latency.

DNS problems are invisible to most tools — applications just hang or fail without explaining why. Wireshark lets you see every DNS query and response, revealing exactly when resolution fails, times out, or returns unexpected results.

## Filter for DNS Traffic

```wireshark
# Show all DNS traffic
dns

# DNS queries only (requests, not responses)
dns.flags.response == 0

# DNS responses only
dns.flags.response == 1

# DNS traffic to/from specific resolver
dns and ip.addr == 8.8.8.8

# DNS queries for a specific name
dns.qry.name == "google.com"
dns.qry.name contains "example"
```

## Reading DNS Packet Details

Expand a DNS packet in the packet detail pane:

```
Domain Name System (query)
  Transaction ID: 0x1234        ← Links query to response
  Flags: 0x0100 Standard query
  Questions: 1
  Queries:
    google.com: type A, class IN  ← Query type: A = IPv4 address

Domain Name System (response)
  Transaction ID: 0x1234        ← Same ID as query = matched response
  Flags: 0x8180 Standard response, No error
  Answers: 1
  Answers:
    google.com: type A, class IN, addr 142.250.80.46
```

## Measure DNS Lookup Latency

Wireshark automatically calculates query response time:

```wireshark
# Show DNS response time field
dns and dns.flags.response == 1

# In packet details:
# Domain Name System (response)
#   [Time: 0.022543000 seconds]  ← DNS lookup time = 22ms

# Sort by response time to find slow queries
# Right-click column header → Sort Ascending/Descending
```

## Find DNS Failures

```wireshark
# NXDOMAIN (non-existent domain)
dns.flags.rcode == 3

# Server failure (SERVFAIL)
dns.flags.rcode == 2

# Query refused
dns.flags.rcode == 5

# Any DNS error
dns.flags.rcode != 0

# All DNS response codes:
# 0 = NOERROR (success)
# 1 = FORMERR (format error)
# 2 = SERVFAIL
# 3 = NXDOMAIN
# 5 = REFUSED
```

## Analyze DNS Response Types

```wireshark
# A records (IPv4 addresses)
dns.qry.type == 1

# AAAA records (IPv6 addresses)
dns.qry.type == 28

# MX records (mail)
dns.qry.type == 15

# PTR records (reverse DNS)
dns.qry.type == 12

# CNAME records (aliases)
dns.qry.type == 5

# TXT records (SPF, DKIM, etc.)
dns.qry.type == 16
```

## Identify Applications Making DNS Queries

```wireshark
# Show all DNS queries with source IPs
dns and dns.flags.response == 0

# In the packet list, the Source column shows which IP is querying
# Sort by Source IP to group by application/host

# Statistics → Conversations
# Shows which hosts are making the most DNS queries
```

## Build DNS Statistics

```
Statistics → DNS (shows breakdown of):
  - Query types (A, AAAA, MX, PTR distribution)
  - Response codes (how many NXDOMAIN, SERVFAIL, etc.)
  - Response times (min, max, average)
  - Most queried names
```

Analyzing DNS in Wireshark gives you ground truth about name resolution — you can see exactly which names are being queried, which resolver is being used, and precisely when lookups fail.
