# How to Capture DNS Query and Response Packets with tcpdump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tcpdump, DNS, Linux, UDP, Networking, Diagnostics

Description: Capture and analyze DNS query and response packets with tcpdump to debug resolution failures, identify slow DNS servers, and monitor DNS traffic patterns.

DNS is the foundation of internet connectivity. When applications fail to reach services, DNS is often the culprit. tcpdump captures the full DNS query/response cycle, showing exactly what names are being resolved, by whom, and how fast.

## Capture All DNS Traffic

```bash
# Capture DNS traffic (UDP port 53)

sudo tcpdump -nn -i eth0 udp port 53

# Include DNS over TCP (zone transfers, large responses)
sudo tcpdump -nn -i eth0 port 53

# DNS to a specific resolver
sudo tcpdump -nn 'port 53 and host 8.8.8.8'

# DNS traffic on all interfaces
sudo tcpdump -nn -i any port 53
```

## Reading DNS tcpdump Output

```bash
sudo tcpdump -nn -v 'udp port 53'

# Query output:
# 10:15:32.123 IP 192.168.1.100.35123 > 8.8.8.8.53: UDP, length 32
#   google.com. A?
#   ^           ^
#   hostname    query type (A = IPv4 address)

# Response output:
# 10:15:32.145 IP 8.8.8.8.53 > 192.168.1.100.35123: UDP, length 68
#   google.com. A 142.250.80.46
#   ^           ^  ^
#   hostname    type  resolved IP

# RTT = 10:15:32.145 - 10:15:32.123 = 22ms (DNS lookup time)
```

## Capture Specific Query Types

```bash
# Capture only A (IPv4) record queries
# DNS query type A = 0x0001, at byte 28 in UDP payload
sudo tcpdump -nn 'udp port 53 and udp[10] & 0x80 = 0'
# "udp[10] & 0x80 = 0" means it's a query (QR bit = 0)

# More readable approach: capture all DNS and grep for types
sudo tcpdump -nn -v 'udp port 53' 2>&1 | grep -E '(A\?|AAAA\?|MX\?|PTR\?|CNAME\?)'
```

## Diagnose DNS Resolution Failures

```bash
# Capture to see if DNS queries are being sent
sudo tcpdump -nn 'udp port 53'

# If you see queries but no responses:
# → DNS server unreachable (check firewall/routing)

# If you see NXDOMAIN in responses:
# → Domain doesn't exist or wrong resolver

# Check verbose response codes
sudo tcpdump -nn -v 'udp port 53' | grep -E '(NXDOMAIN|SERVFAIL|REFUSED)'
```

## Monitor Which Domains an Application Resolves

```bash
# Capture DNS queries from a specific source IP
sudo tcpdump -nn -v 'src 192.168.1.50 and udp port 53'

# Find all unique domains being resolved
sudo tcpdump -nn -v 'udp port 53' 2>/dev/null | \
  grep -oP '\s+\K\S+(?= A\?)' | sort -u

# Find the most queried domains
sudo tcpdump -nn -v -c 500 'udp port 53' 2>/dev/null | \
  grep -oP '\s+\K\S+(?= A\?)' | sort | uniq -c | sort -rn | head -20
```

## Save DNS Capture for Audit

```bash
# Capture 10 minutes of DNS activity
sudo timeout 600 tcpdump -nn -i eth0 'port 53' -w /tmp/dns-audit.pcap

# Analyze: find all resolved domains
sudo tcpdump -nn -v -r /tmp/dns-audit.pcap | grep 'A?' | \
  awk '{print $NF}' | sort -u

# Find DNS queries that got no response (timeouts)
sudo tcpdump -nn -r /tmp/dns-audit.pcap | \
  awk '/A\?/ {q[$1]=$0} END {for(t in q) print q[t]}'
```

## Check DNS Response Time Distribution

```bash
#!/bin/bash
# dns-latency.sh - Measure DNS resolution times

echo "Measuring DNS latency to 8.8.8.8..."

for i in $(seq 1 5); do
    START=$(date +%s%3N)
    # Force fresh DNS query (no caching)
    dig @8.8.8.8 +norecurse google.com > /dev/null 2>&1
    END=$(date +%s%3N)
    echo "Query $i: $((END-START))ms"
done
```

Capturing DNS traffic with tcpdump is the definitive way to debug resolution problems - if you can see queries going out and responses coming back, DNS is working; if you see queries but no responses, you've found your problem.
