# How to Fix TCP Connection Reset by Peer Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, RST, Reset, Connection, Debugging, Linux, Networking

Description: Diagnose and fix TCP 'connection reset by peer' errors by identifying whether the RST originates from the application, a proxy, a firewall, or the kernel.

## Introduction

"Connection reset by peer" means a TCP RST segment was received mid-connection. Unlike a graceful FIN (which is normal connection close), a RST abruptly terminates the connection discarding any data in flight. RSTs come from several sources: the peer application crashing or explicitly calling RSO_LINGER, a proxy or load balancer with a shorter idle timeout, a firewall rule, or the Linux kernel's RST generation for invalid packets.

## Identify the RST Source

```bash
# Capture on the interface to see who sends the RST

tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-rst != 0' -w /tmp/rst.pcap

# Analyze: who is the source IP of the RST?
tcpdump -r /tmp/rst.pcap -n 'tcp[tcpflags] & tcp-rst != 0' | \
  awk '{print $3}' | sort | uniq -c | sort -rn
# Source IP of RST = entity sending the reset
```

## RST Sources and Fixes

### 1. Application Crashes or Exits Abruptly

```bash
# The remote app process died while the connection was open
# RST comes from the remote host's kernel (socket closed without FIN)

# Check if the remote service is stable:
systemctl status remote-service
journalctl -u remote-service --since "5 minutes ago" | tail -20

# Fix: Ensure proper graceful shutdown in the application
# In Python: socket.shutdown(SHUT_RDWR) before socket.close()
# In Java: socket.shutdownOutput() before socket.close()
```

### 2. Proxy or Load Balancer Idle Timeout

```bash
# Proxy kills idle connections and sends RST
# Symptom: RST appears after exact idle period (e.g., 60 seconds)

# Check proxy timeout config (nginx example):
grep -E "keepalive_timeout|proxy_read_timeout|proxy_connect_timeout" /etc/nginx/nginx.conf

# Fix options:
# 1. Enable TCP keepalives in your application to keep connection active
sysctl -w net.ipv4.tcp_keepalive_time=30    # Start keepalives after 30s idle
sysctl -w net.ipv4.tcp_keepalive_intvl=10   # Probe every 10s
sysctl -w net.ipv4.tcp_keepalive_probes=3   # 3 probes before drop

# 2. Increase proxy idle timeout to match your application's idle pattern
# 3. Implement connection pool keepalive in application code
```

### 3. Firewall Stateful Table Expiry

```bash
# Stateful firewall drops connection from its table (memory pressure or timeout)
# Then when more packets arrive for that connection: RST or drop

# Check conntrack table size:
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max

# If count is close to max: table exhausted
# Fix: increase max or reduce timeout values
sysctl -w net.netfilter.nf_conntrack_max=262144
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=600  # 10 minutes

# Check for RST from firewall host:
iptables -n -L -v | grep REJECT
# REJECT sends RST; DROP silently drops
# If peer receives RST from your firewall: RST comes from firewall host IP
```

### 4. RSO_LINGER with Linger=0

```bash
# Some applications set SO_LINGER with l_linger=0
# This causes immediate RST on socket close (no graceful FIN)
# Common in web servers to quickly reclaim resources

# Check if application uses SO_LINGER:
strace -p $(pgrep myapp) -e trace=setsockopt 2>&1 | grep LINGER

# In application code (Python fix):
# Remove: s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', 1, 0))
# Use graceful close instead
```

### 5. SEQ Number Out of Window

```bash
# Linux kernel sends RST if it receives a packet with SEQ outside the window
# Causes: NAT table expiry, connection reuse after crash, TCP sequence wrap

# Check for these RSTs in kernel stats:
nstat | grep -i "outofwindow\|outside"

# Fix: ensure proper connection close before reuse
# For persistent connections: validate connection is alive before reuse
# Use connection health checks in connection pools
```

## Application-Level Fixes

```bash
# 1. Handle reset errors gracefully (retry logic)
# In any language: catch ECONNRESET, retry with exponential backoff

# 2. Enable TCP keepalives at application level (most robust fix)
# Python example:
# import socket
# sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
# sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 30)
# sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)
# sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 3)
```

## Conclusion

TCP connection resets always have a sender. Use tcpdump to identify the RST source IP - that tells you which component is sending it. Application crashes cause RSTs from the remote host. Proxy/firewall timeouts cause RSTs after an idle period from the proxy/firewall IP. Kernel RSTs come from the peer host itself due to invalid sequence numbers or linger settings. Fix at the source: configure keepalives for timeout-related resets, fix application shutdown for crash-related resets, and tune firewall tables for conntrack exhaustion.
