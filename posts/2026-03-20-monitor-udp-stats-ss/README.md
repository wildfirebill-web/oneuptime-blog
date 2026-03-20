# How to Monitor UDP Connection Statistics with ss

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, ss, Monitoring, Linux, Socket Statistics, Networking

Description: Use the ss command to monitor UDP socket statistics, view buffer usage, identify packet drops, and inspect UDP socket state on Linux.

## Introduction

The `ss` command provides socket-level statistics for UDP in a way that `netstat` cannot match in detail or speed. While UDP is connectionless, the kernel still maintains socket state: bound address, port, buffer sizes, and drop counters. `ss` exposes all of this per socket, making it the right tool to verify UDP services are listening, check buffer utilization, and identify which sockets are dropping packets.

## Basic UDP Socket Listing

```bash
# List all UDP sockets

ss -un

# List with process info (requires root for other users' processes)
ss -unp

# List listening UDP sockets only
ss -ulnp

# List all UDP sockets (including non-listening)
ss -una

# Sample output:
# State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
# UNCONN  0       0       0.0.0.0:5000       0.0.0.0:*          users:(("server",pid=1234,fd=5))
# UNCONN  0       0       0.0.0.0:53         0.0.0.0:*          users:(("dnsmasq",pid=567,fd=6))
```

## UDP States

```text
UDP socket states in ss:
- UNCONN: Socket is bound but not connected (listening for any sender)
         Most UDP servers are in this state

- ESTAB:  Socket is "connected" to a specific remote addr/port
         (not a real connection; just the kernel's default destination)
         Created with sock.connect() in application

Recv-Q: Bytes in receive buffer waiting to be read by application
        > 0 consistently means application is not reading fast enough

Send-Q: Bytes in send buffer waiting to be sent
        > 0 means network send path is congested
```

## View Buffer Usage and Drops

```bash
# Show memory usage (buffer info) for UDP sockets
ss -umn

# Output includes:
# skmem:(r0,rb212992,t0,tb212992,f0,w0,o0,bl0,d0)
# r = receive bytes currently queued
# rb = receive buffer size
# t = transmit bytes currently queued
# tb = transmit buffer size
# d = drops (packets dropped due to full buffer)

# Filter for sockets with drops:
ss -umn | grep -v 'd0'
# Shows sockets where d != 0 (drops occurred)

# Monitor buffer fill level over time:
while true; do
    ss -umn sport = :5000 | grep -oP 'r\K[0-9]+(?=,)'
    sleep 1
done
```

## Filter UDP by Port and Address

```bash
# Show UDP sockets on specific port
ss -unp sport = :5000

# Show UDP sockets connected to specific destination
ss -unp dst 10.20.0.5

# Show UDP sockets from specific source
ss -unp src 10.20.0.1

# Combine filters:
ss -unp src 10.20.0.1 dst 10.20.0.5

# Show UDP on a port range:
ss -unp sport '>= :5000' sport '<= :5100'
```

## Check for High Receive Queue

```bash
# Recv-Q > 0 for UDP = application not reading fast enough
# If Recv-Q stays high: packets will soon be dropped (buffer full)

# Watch Recv-Q in real-time:
watch -n 0.5 "ss -unp sport = :5000 | grep 5000"

# Check global UDP receive errors:
nstat | grep UdpRcvbufErrors
# Correlate with Recv-Q being high

# Alert if any UDP socket has high receive queue:
ss -umn | awk -F'[,(]' '/UNCONN/{
    for(i=1;i<=NF;i++) {
        if ($i ~ /^r[0-9]+$/) {
            bytes = substr($i, 2)
            if (bytes+0 > 100000) printf "HIGH RQ: %s bytes on %s\n", bytes, $5
        }
    }
}'
```

## Compare ss vs netstat for UDP

```bash
# netstat (older, slower, but familiar):
netstat -un    # UDP sockets
netstat -su    # UDP statistics summary

# ss (modern, faster, more detail):
ss -un         # UDP sockets
ss -s          # Summary including UDP stats

# Summary statistics:
ss -s
# Shows:
# UDP: total N
# (Unlike netstat -s, ss -s doesn't show drop counts in summary)
# Use nstat for UDP drop counters
```

## Conclusion

`ss -umn` is the definitive command for UDP socket monitoring on Linux: it shows buffer usage, queue depths, and drop counts per socket. Watch `Recv-Q > 0` as an early warning that a UDP service is falling behind. The `d` counter in `skmem` shows socket-level drops. Combine with `nstat | grep Udp` for system-wide drop statistics. Together, these tools identify exactly which UDP socket is dropping packets and whether the cause is buffer size or application processing speed.
