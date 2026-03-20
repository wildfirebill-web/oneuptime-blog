# How to Fix TCP SYN Timeout Issues on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Linux, Networking, SYN, Timeout, Kernel

Description: Diagnose and fix TCP SYN timeout problems on Linux by tuning kernel retransmission parameters, SYN backlog queues, and SYN cookie protection.

## Introduction

A TCP SYN timeout occurs when a client sends a SYN packet but never receives a SYN-ACK, causing the connection attempt to fail after multiple retransmissions. On Linux, the kernel controls how many times it retransmits SYN before giving up, and how long it waits between retries. Tuning these parameters balances connection reliability against resource usage.

## Understanding SYN Retransmission

Linux retransmits SYN packets using exponential backoff:

```text
SYN attempt 1: wait 1 second
SYN attempt 2: wait 2 seconds
SYN attempt 3: wait 4 seconds
SYN attempt 4: wait 8 seconds
SYN attempt 5: wait 16 seconds
SYN attempt 6: wait 32 seconds
Total: ~63 seconds before "Connection timed out"
```

## Viewing and Adjusting SYN Retry Count

```bash
# View current SYN retry count (default: 6)

sysctl net.ipv4.tcp_syn_retries
# net.ipv4.tcp_syn_retries = 6

# Reduce retries to fail faster (useful for services that quickly need to know)
# 3 retries = 1+2+4+8 = ~15 second timeout
sysctl -w net.ipv4.tcp_syn_retries=3

# For servers responding to inbound SYNs (SYN-ACK retries)
sysctl net.ipv4.tcp_synack_retries
# Default: 5

# Reduce SYN-ACK retries to free resources faster during SYN floods
sysctl -w net.ipv4.tcp_synack_retries=3
```

## SYN Backlog Queue Configuration

The SYN backlog queue holds half-open connections waiting for the final ACK:

```bash
# View current SYN backlog size
sysctl net.ipv4.tcp_max_syn_backlog
# Default: 128 (too low for busy servers)

# Increase for high-traffic services
sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# Application-level backlog (second argument to listen())
# Python example:
# socket.listen(1024)  # Increase from default
```

## SYN Cookies (Protection Against SYN Floods)

SYN cookies allow the server to avoid allocating state for each SYN, defeating SYN flood attacks:

```bash
# Check if SYN cookies are enabled
sysctl net.ipv4.tcp_syncookies
# 1 = enabled when queue overflows (default)
# 2 = always enabled (more secure, slight CPU overhead)

# Enable SYN cookies permanently
sysctl -w net.ipv4.tcp_syncookies=1
echo "net.ipv4.tcp_syncookies=1" >> /etc/sysctl.conf
```

## Diagnosing SYN Timeouts

```bash
# Check for SYN timeout counters
netstat -s | grep "SYNs to LISTEN sockets dropped"

# Check kernel messages for SYN flood warnings
dmesg | grep "SYN flooding"
# "Possible SYN flooding on port 80. Sending cookies."

# Watch active SYN_SENT connections
watch -n 1 "ss -tn state syn-sent | wc -l"

# Capture SYN retransmissions
tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-syn != 0' | \
  awk '{count[$3]++; if (count[$3]>2) print "Multiple SYNs from: "$3}'
```

## Application-Level SYN Timeout Configuration

```python
import socket

# Set connection timeout in application code
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.settimeout(5)  # Fail after 5 seconds if no SYN-ACK received

try:
    s.connect(('10.20.0.5', 8080))
except socket.timeout:
    print("Connection timed out - no SYN-ACK received within 5 seconds")
except ConnectionRefusedError:
    print("Connection refused - RST received")
```

## Conclusion

TCP SYN timeouts are controlled by kernel parameters that balance reliability with resource efficiency. Reduce `tcp_syn_retries` to fail faster in environments where quick failure detection matters. Increase `tcp_max_syn_backlog` for high-traffic servers. Always enable SYN cookies to protect against SYN flood attacks. Application-level timeouts should be significantly shorter than the kernel's default 63-second timeout.
