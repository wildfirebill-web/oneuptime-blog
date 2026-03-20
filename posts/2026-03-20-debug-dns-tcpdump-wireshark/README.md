# How to Debug DNS Resolution with tcpdump and Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, tcpdump, Wireshark, Debugging, Packet Capture, Networking

Description: Capture and analyze DNS traffic with tcpdump and Wireshark to trace resolution failures, measure query latency, and identify malformed or blocked DNS responses.

## Introduction

When `dig` shows the wrong answer or DNS is failing, packet capture reveals exactly what's happening at the wire level. `tcpdump` and Wireshark show whether queries are leaving the host, whether responses arrive, what the response contains, and whether DNSSEC validation is causing failures. This approach is definitive - it shows reality, not what the application thinks is happening.

## Capture DNS Traffic with tcpdump

```bash
# Capture all DNS traffic (UDP and TCP):

tcpdump -i eth0 -n 'port 53'

# Save to file for analysis:
tcpdump -i eth0 -n 'port 53' -w /tmp/dns.pcap

# Verbose output showing DNS details:
tcpdump -i eth0 -n -v 'port 53' 2>/dev/null | head -50
# -v shows DNS record types, names, TTLs

# Filter specific domain:
tcpdump -i eth0 -n -v 'port 53 and (udp or tcp)' 2>/dev/null | \
  grep -i "example.com"

# Show DNS queries only (not responses):
# DNS QR bit: 0 = query, 1 = response
# In BPF: dns[2] & 0x80 = 0 for queries
tcpdump -i eth0 -n 'udp port 53 and udp[10] & 0x80 = 0'

# Show DNS responses only:
tcpdump -i eth0 -n 'udp port 53 and udp[10] & 0x80 != 0'
```

## Diagnose Common DNS Failures

```bash
# Case 1: Query sent, no response
# Capture while doing dig:
tcpdump -i eth0 -n 'port 53' -c 20 &
dig @8.8.8.8 example.com
# If you see query packets but no response: resolver not responding
# → Check firewall rules, resolver availability

# Case 2: Query sent, NXDOMAIN response
tcpdump -i eth0 -n -v 'port 53' 2>/dev/null | grep -E "NXDOMAIN|NOERROR"
# NXDOMAIN → domain doesn't exist at queried server
# Check: is this a split-horizon issue? Query right resolver?

# Case 3: TCP instead of UDP (unexpected)
tcpdump -i eth0 -n 'tcp port 53' -c 20
# If there are TCP connections: something is causing TCP fallback
# Common causes: large responses, UDP blocked, resolver prefers TCP

# Case 4: SERVFAIL responses
tcpdump -i eth0 -n -v 'port 53' 2>/dev/null | grep -i "SERVFAIL"
# SERVFAIL from resolver → resolver couldn't get authoritative answer
# Could be DNSSEC validation failure or upstream server issue
```

## Wireshark DNS Analysis

```text
Wireshark display filters for DNS:

# Show all DNS:
dns

# Show DNS queries only:
dns.flags.response == 0

# Show DNS responses:
dns.flags.response == 1

# Filter by response code:
dns.flags.rcode == 0    # NOERROR (success)
dns.flags.rcode == 2    # SERVFAIL
dns.flags.rcode == 3    # NXDOMAIN
dns.flags.rcode == 5    # REFUSED

# Filter by query type:
dns.qry.type == 1       # A records
dns.qry.type == 28      # AAAA records
dns.qry.type == 15      # MX records

# Filter by domain:
dns.qry.name contains "example.com"

# Show slow DNS (query time > 100ms):
dns.time > 0.1
```

## Extract DNS Statistics from Capture

```bash
# tshark (command-line Wireshark) for statistics:

# DNS query counts by domain:
tshark -r /tmp/dns.pcap -Y 'dns.flags.response == 0' \
  -T fields -e dns.qry.name | sort | uniq -c | sort -rn | head -20

# DNS response time statistics:
tshark -r /tmp/dns.pcap -Y 'dns' -T fields \
  -e frame.time_relative -e dns.qry.name -e dns.time \
  | grep -v '^\s*$' | head -20

# Find slow DNS responses (> 200ms):
tshark -r /tmp/dns.pcap -Y 'dns.time > 0.2' \
  -T fields -e dns.qry.name -e dns.time

# DNS response codes summary:
tshark -r /tmp/dns.pcap -Y 'dns.flags.response == 1' \
  -T fields -e dns.flags.rcode | sort | uniq -c
# 0 = NOERROR, 2 = SERVFAIL, 3 = NXDOMAIN
```

## Trace a Specific Resolution

```bash
# Start capture and immediately run dig:
tcpdump -i eth0 -n -v 'port 53' -w /tmp/dig_trace.pcap &
TCPDUMP_PID=$!
dig +trace +stats example.com > /tmp/dig_output.txt 2>&1
sleep 2
kill $TCPDUMP_PID

# Analyze the capture:
tshark -r /tmp/dig_trace.pcap -Y 'dns' \
  -T fields -e ip.src -e ip.dst -e dns.qry.name -e dns.flags.response

echo "=== dig output ==="
cat /tmp/dig_output.txt
```

## Conclusion

tcpdump and Wireshark reveal the ground truth about DNS: whether packets are sent, whether responses arrive, and what they contain. For failures, check whether the query packet appears on the wire (if not: local firewall or resolver not starting). Check whether a response arrives (if not: resolver unreachable or firewall blocking). Check the response code in the answer (NXDOMAIN, SERVFAIL, REFUSED each indicate different problems). `tshark -Y 'dns.time > 0.1'` quickly identifies slow DNS resolvers from captured traffic.
