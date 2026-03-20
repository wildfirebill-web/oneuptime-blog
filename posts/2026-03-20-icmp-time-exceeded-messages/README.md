# How to Understand ICMP Time Exceeded Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMP, Networking, Traceroute, IPv4, TTL, Troubleshooting

Description: Understand ICMP Type 11 Time Exceeded messages, how they enable traceroute to work, and what they reveal about routing loops and fragmentation problems.

## Introduction

ICMP Time Exceeded (Type 11) is generated when a router decrements a packet's TTL to zero. This built-in anti-loop mechanism prevents packets from circulating forever. The Time Exceeded message includes the original packet's header, giving the source enough information to identify the problem. Traceroute exploits this mechanism deliberately.

## ICMP Type 11 Codes

| Code | Name | When Generated |
|---|---|---|
| 0 | TTL Exceeded in Transit | Router decrements TTL to 0 |
| 1 | Fragment Reassembly Time Exceeded | Destination can't reassemble all fragments in time |

## How Traceroute Uses TTL Exceeded

```bash
# Traceroute sends probes with TTL=1, 2, 3, etc.
# Each router sends back ICMP Type 11 Code 0 when it decrements TTL to 0
# This reveals each hop's IP address and RTT

# Capture traceroute ICMP exchanges
tcpdump -i eth0 -n '(icmp[0] = 11) or (udp dst portrange 33434-33534)'
```

## Capturing Time Exceeded Messages

```bash
# Watch for TTL exceeded messages in real time
tcpdump -i eth0 -n -v 'icmp[0] = 11 and icmp[1] = 0'

# Example output:
# IP 10.0.0.1 > 192.168.1.10: ICMP time exceeded in-transit
# -> Router 10.0.0.1 discarded our probe because TTL hit 0 there
```

## Using TTL to Detect Routing Loops

```bash
# If traceroute shows the same IP repeating, there's a routing loop
traceroute -n 10.20.0.5

# Example loop:
# 3  10.0.0.1  2.3 ms
# 4  10.0.1.1  2.8 ms
# 5  10.0.0.1  2.4 ms   <- loop!
# 6  10.0.1.1  2.9 ms
# ...

# Detect loop in traceroute output
traceroute -n 10.20.0.5 | awk '{print $2}' | sort | uniq -d
# If any IP appears more than once, it's in a loop
```

## Fragment Reassembly Timeout (Code 1)

Code 1 is generated when a destination receives some fragments of a packet but not all, and the reassembly timer expires (default 60 seconds):

```bash
# Monitor for fragment reassembly failures
tcpdump -i eth0 -n 'icmp[0] = 11 and icmp[1] = 1'

# This often appears in VPN tunnels or networks with inconsistent MTU
# Fix: reduce MTU or fix PMTUD so fragmentation is avoided
```

## TTL Values and Network Distance

```bash
# Most OSes start with TTL 64 (Linux) or 128 (Windows)
# The received TTL tells you roughly how many hops away the host is

ping -c 4 8.8.8.8 | grep ttl
# ttl=118 -> 128 - 118 = 10 hops away (Windows starting TTL)
# ttl=54  -> 64 - 54 = 10 hops away (Linux starting TTL)

# Set custom TTL to trace a specific hop
ping -t 3 8.8.8.8   # macOS: -m on Linux
# Response from hop 3's router
```

## Conclusion

ICMP Time Exceeded messages are the backbone of traceroute functionality and a signal of routing loops. Code 0 (TTL in transit) is expected during traceroute and indicates a routing loop when the same router appears repeatedly. Code 1 (fragment reassembly) signals MTU or fragmentation problems. Both types deserve attention in packet captures as they point directly to infrastructure issues.
