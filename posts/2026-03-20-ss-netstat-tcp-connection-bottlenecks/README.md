# How to Use ss and netstat to Identify TCP Connection Bottlenecks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ss, netstat, TCP, Linux, Troubleshooting, Connection State, Network Monitoring

Description: Learn how to use the ss and netstat commands to identify TCP connection bottlenecks, including connection state counts, socket queues, retransmits, and port exhaustion.

---

`ss` (socket statistics) is the modern replacement for `netstat`, offering faster output and more detail. Both tools are essential for diagnosing TCP connection bottlenecks.

## Basic Connection Overview

```bash
# Summary of all socket states
ss -s

# Output:
# Total: 1234
# TCP:   456 (estab 200, closed 10, orphaned 2, timewait 244)
#
# Transport  Total  IP   IPv6
# RAW        0      0    0
# UDP        12     8    4
# TCP        456    400  56

# netstat equivalent
netstat -s | grep -E "connections|failed|reset"
```

## Counting Connections by State

```bash
# Count TCP connections by state
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# Output:
#  244 TIME-WAIT
#  200 ESTABLISHED
#   10 CLOSE-WAIT
#    5 SYN-SENT

# netstat equivalent
netstat -tan | awk 'NR>2 {print $6}' | sort | uniq -c | sort -rn
```

## Identifying Listen Queue Backlog

```bash
# Show listen queue: Recv-Q is queued but not accepted
ss -tlnp

# Output columns: State, Recv-Q, Send-Q, Local Address, Peer Address, Process
# LISTEN  0  128  0.0.0.0:80  0.0.0.0:*  users:(("nginx",pid=1234,fd=6))

# Recv-Q approaching the backlog limit (128) indicates accept() bottleneck
# Increase with: sysctl -w net.core.somaxconn=65535
```

## Finding Connections with Large Send/Receive Queues

```bash
# High Send-Q means data waiting to be sent (slow client or network)
ss -tn | awk 'NR>1 && $2>0' | head    # Non-zero Recv-Q
ss -tn | awk 'NR>1 && $3>0' | head    # Non-zero Send-Q

# Sort by send queue size
ss -tn | sort -k3 -rn | head -20
```

## Viewing Per-Connection Details (Retransmits, RTT)

```bash
# Show extended info: retransmits, RTT, congestion window
ss -tni | grep -A1 ESTAB | grep -v ESTAB | head -40

# Example output:
# cubic rto:204 rtt:0.5/0.25 ato:40 mss:1460 pmtu:1500 rcvmss:1460 \
#   advmss:1460 cwnd:10 bytes_acked:12345 retrans:0/0 ...
```

## Detecting Port Exhaustion

```bash
# Check ephemeral port range
sysctl net.ipv4.ip_local_port_range
# Default: 32768 60999  (28231 ports)

# Count TIME_WAIT connections (ports stuck in TIME_WAIT are unavailable)
ss -tan state time-wait | wc -l

# If count is near 28000, you're approaching port exhaustion
# Fix: increase port range or enable tw_reuse
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
sysctl -w net.ipv4.tcp_tw_reuse=1
```

## Identifying Connections by Process

```bash
# Show which process owns each connection (requires root)
ss -tnp | grep :80

# Output:
# ESTAB 0 0 10.0.0.1:80 10.0.0.50:45678 users:(("nginx",pid=1234,fd=12))

# netstat equivalent
netstat -tnp | grep :80
```

## Quick Bottleneck Diagnostic Script

```bash
#!/bin/bash
echo "=== Socket Summary ==="
ss -s

echo ""
echo "=== TCP States ==="
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

echo ""
echo "=== Listen Backlogs (Recv-Q > 0) ==="
ss -tlnp | awk 'NR>1 && $2>0'

echo ""
echo "=== High Send-Q Connections ==="
ss -tn | awk 'NR>1 && $3>10000' | head -10
```

## Key Takeaways

- Use `ss -s` for a quick overview; look for high TIME_WAIT or CLOSE_WAIT counts.
- A high Recv-Q on a LISTEN socket indicates the application is not accepting connections fast enough.
- Use `ss -tni` to see per-connection retransmit counts and RTT — high retransmits indicate packet loss.
- TIME_WAIT exhaustion causes `connect: Cannot assign requested address`; fix with `tcp_tw_reuse` and wider port ranges.
