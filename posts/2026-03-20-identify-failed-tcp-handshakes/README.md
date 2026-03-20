# How to Identify Failed TCP Handshakes in Packet Captures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Wireshark, Networking, Troubleshooting, Packet Analysis

Description: Use packet captures and Wireshark filters to identify incomplete TCP handshakes, distinguish between unanswered SYNs, rejected connections, and RST-terminated handshakes.

## Introduction

A failed TCP handshake means a connection never reached the ESTABLISHED state. The failure mode - unanswered SYN, RST response, or SYN-ACK never followed by ACK - tells you where to look for the problem. Packet captures provide definitive proof of what happened at the transport layer.

## Types of Failed Handshakes

### Type 1: Unanswered SYN (Timeout)

The client sends SYN but never receives SYN-ACK. The OS retransmits SYN several times before giving up.

```bash
# Identify unanswered SYNs: SYN sent, retransmitted multiple times, no SYN-ACK

tcpdump -n 'tcp[tcpflags] & tcp-syn != 0'

# Signs: same SYN packet appears 3-6 times with no intervening SYN-ACK
# Cause: server down, firewall silently dropping SYN, routing failure
```

Wireshark filter:
```text
# Find SYN retransmissions
tcp.analysis.retransmission && tcp.flags.syn == 1 && tcp.flags.ack == 0
```

### Type 2: RST Response to SYN

Server receives SYN but sends RST instead of SYN-ACK - connection actively refused.

```bash
# Capture: SYN from client → RST from server
tcpdump -n 'tcp[tcpflags] & tcp-rst != 0 and host 10.20.0.5'

# Cause: port closed, application not listening, or iptables REJECT rule
```

Wireshark filter:
```text
# Find RSTs that come right after SYNs (same stream)
tcp.flags.reset == 1
```

### Type 3: SYN-ACK Sent but No ACK

Server receives SYN and sends SYN-ACK, but the client never sends the final ACK. This is a half-open connection, often seen in SYN flood attacks.

```bash
# Watch for SYN-RECEIVED connections that never complete
ss -tn state syn-received

# Count half-open connections
ss -tn state syn-received | wc -l
```

Wireshark filter:
```text
# Find SYN-ACK packets - check if matching ACK follows
tcp.flags.syn == 1 && tcp.flags.ack == 1
```

## Automated Detection with tcpdump

```bash
# Capture SYNs and flag any that are retransmitted (unanswered)
tcpdump -i eth0 -n -w /tmp/syns.pcap 'tcp[tcpflags] & tcp-syn != 0' &
sleep 60
kill %1

# Analyze: find source IPs with multiple SYN retransmits to same port
tshark -r /tmp/syns.pcap -T fields \
  -e ip.src -e ip.dst -e tcp.dstport \
  | sort | uniq -c | sort -rn | head -20
```

## Using ss to Find Failed Handshakes

```bash
# SYN_SENT: client waiting for SYN-ACK (handshake in progress)
ss -tn state syn-sent

# SYN_RECV: server waiting for ACK (half-open)
ss -tn state syn-received

# If many entries persist here: handshake failures are occurring
# SYN_RECV backlog full = SYN flood or overloaded server
```

## Kernel Metrics for Failed Handshakes

```bash
# AttemptFails: TCP connections that failed (SYN sent but never completed)
cat /proc/net/snmp | awk '/^Tcp:/{getline; print}' | tr ' ' '\n' | grep -A1 AttemptFails

# Or use nstat
nstat | grep TcpAttemptFails

# High AttemptFails indicates:
# - Many connections being refused (RST response)
# - Or many connections timing out (no response)
```

## Conclusion

Failed TCP handshakes leave clear traces in packet captures. Unanswered SYNs with retransmissions point to server/firewall issues. Immediate RSTs point to closed ports or REJECT firewall rules. Persistent SYN-RECEIVED states indicate half-open connections from SYN floods or client-side bugs. Combining `ss` state filters with tcpdump captures gives you both real-time visibility and forensic analysis capability.
